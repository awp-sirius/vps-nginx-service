# Шпаргалка для поднятия сервиса на VPS.
_На примере web API ASP .NET Core (.Net 6)_


- [Настройка домена](#SetDomain)
- [Первоначальная конфигурация, установка Docker](#DockerConf)
- [Запуск необходимых контейнеров](#RunContainers)
  - [mongoDB](#MongoDb)
  - [Service API](#ServiceAPI)
- [Настройка Nginx, SSL](#SetNginxSSL)
  - [Nginx](#Nginx)
  - [Certbot](#Cerbot)
  - [Перевыпуск сертификата](#CerbotRenew)
- [Итого](#Results)
- [Заметки](#Notes)

<a id="SetDomain"></a>
## Настройка домена
Необходимо прописать IP VPS в A-записи DNS для {domain} и www.{domain}

<a id="DockerConf"></a>
## Первоначальная конфигурация, установка Docker.
Подключение из powershell по ssh с паролем (не безопасно, лучше настроить ssh токен):
```bash
ssh -l root {ip/domain}
```
Сменить пароль root
```bash
sudo passwd root
```
Для Ubuntu:
* Обновляем пакеты
  ```bash
  sudo apt-get update
  ```
* Устанавливаем необходимое для докера
  ```bash
  sudo apt-get install ca-certificates
  sudo apt-get install curl
  ```
  Возможно, потребуется парезапуск
  ```bash
  sudo apt-get install gnupg
  sudo apt-get install lsb-release
  ```
* Ключи для докера
  ```bash
  sudo mkdir -p /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  ```
  ```bash
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  ```
* Устанавливаем сам докер
  ```bash
  sudo apt-get update
  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
  ```
* Добавляем пользователя
  ```bash
  sudo usermod -aG docker $(whoami)
  ```
* Проверяем статус
  ```bash
  sudo systemctl status docker
  ```
* Установка портов
  ```bash
  sudo iptables -A INPUT -i lo -j ACCEPT
  sudo iptables -A OUTPUT -o lo -j ACCEPT 
  sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
  sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
  sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
  ```
  
<a id="RunContainers"></a>
## Запуск необходимых контейнеров
Я сознательно не использую docker compose.
Для работы сервиса мне потребуется mongoDB и сам image сервиса.

Для начала поднимем внутренний мост для связи между контейнерами.
```bash
docker network create {bridgeName} --driver bridge
```
<a id="MongoDb"></a>
### mongoDB
Запуск монги для теста:
```bash
docker run -d -p 27017:27017 --name {mongoName} mongo:latest
```
ConnectionString для подключения: `mongodb://{ip vps}:27017/`
* Чтобы данные не удалялись после перезапуска контейнера, необходимо использоваль хранилище volume:

  `
  -v {mongoVolumeName}:/data/db
  `
* Чтобы получить доступ в монгу из нашего сервиса API (из другого контейнера), необходимо использовать мост:

  `
  --network {bridgeName}
  `
* Чтобы создать root пользователя при инициализации, можно передать переменные среды:

  `
  -e MONGO_INITDB_ROOT_USERNAME="{rootUserName}"
  `

  `
  -e MONGO_INITDB_ROOT_PASSWORD="{rootUserPassword}"
  `

Удалить тестовый контейнер:
```bash
docker rm -f {mongoName}
```

* Убираем открытый порт 27017 и запускаем монгу:
```bash
docker run -d -v {mongoVolumeName}:/data/db -e MONGO_INITDB_ROOT_USERNAME={rootUserName} -e MONGO_INITDB_ROOT_PASSWORD={rootUserPassword} --restart always --network {bridgeName} --name {mongoName} mongo:latest
```

* Создаём отдельного пользователя для базы данных:
```bash
docker exec -it mongodb mongosh -u {rootUserName} -p {rootUserPassword}
```
```mongosh
use mydb
db.createUser({user: "myuser", pwd: "mypassword", roles: [{ role: 'readWrite', db:'mydb' }]})
```
Здесь `use mydb` переключает на базу данных mydb и создаёт её, если её не существует.

`db.createUser` создаёт нового пользователя с правами доступа, согласно roles.

Однако, сам пользователь будет создан в той бд, где мы находимся (при авторизации нужно будет указать mydb). Как вариант: `use admin`.

`mongosh` жрёт прилично оперативной памяти.

<a id="ServiceAPI"></a>
### Service API
В Dockerfile сервиса убираю EXPOSE 443, т.к. в {bridgeName} трафик будет ходить без ssl.

В Program/Config убираю app.UseHttpsRedirection(), дабы не редиректило.

Публикуем image и запускаем контейнер:
```bash
docker run -d -e MONGO_USER_PASSWORD="{mypassword}" -e MONGO_USER_NAME="{myuser}" -e MONGO_HOST="{mongoName}" -e MONGO_PORT="{port}" --restart always --network {bridgeName} --name {serviceName} {dockerImageName}
```

Считывание переменных из среды:
```C#
var userPassword = Environment.GetEnvironmentVariable("MONGO_USER_PASSWORD");
```

Для теста можно было открыть 80 порт tcp:

`
-p 80:80/tcp
`

<a id="SetNginxSSL"></a>
## Настройка Nginx, SSL
<a id="Nginx"></a>
### Nginx
Для nginx'a нам нужно будет расшарить 3 дирректории с использованием bind.
* Bind для конфигурации `/etc/nginx/conf.d/default.conf`
* Bind для получения и проверки сертификата Let's Encrypt `/var/www/certbot`
* Bind для приватного и публичного ключа сертификата `/etc/letsencrypt`

Можно использовать как `-v` так и `--mount`. Однако, `-mount` не создаст дирректорию, если её не существует. А '-v' автоматически определяет bind или volume (если путь - то bind, если строка - то volume.

Запуск nginx
```bash
docker run -d -p 80:80 -p 443:443 -v ~/certbot/www:/var/www/certbot:rw -v ~/certbot/conf:/etc/letsencrypt:rw -v ~/NginxConf:/etc/nginx/conf.d:ro --restart always --network {bridgeName} --name {nginxName} nginx
```
Здесь будут созданы привязки bind, а не volume

<a id="Cerbot"></a>
### Certbot 
Я буду использовать отдельный контейнер с image certbot, но перед получением сертификата необходимо подготовиться.

1. Изменить default.conf для обработки запросов Let's Encrypt:
  * Открываем редактор файла:
```bash
vi ~/NginxConf/default.conf
```
  * Нажимаем INSERT для редактирования
  * Для удаления всех строк нужно нажать [ESC], ввести `:%d`, нажать [Enter].
  * Вставляем конфиг. **Обязательно** заполняем оба server_name.
```conf
server {
    listen 80;
    listen [::]:80;

    server_name  {domain} www.{domain};
    server_tokens off;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}
```
  * Проверяем т.к. может потеряться первый символ
  * Для сохранения: [ESC], ввести `:wq`, нажать [Enter].
  * Для выхода без сохранения: [ESC], ZZ (Shift + Z или со включённым CapsLock)
2. Перезапустить nginx
```bash
docker restart {nginxName}
```
3. Получаем сертификат
Я буду использовать команду, которую можно передать при запуске контейнера:

`
certonly --webroot -w /var/www/certbot --force-renewal --email {yourEmail} -d {domain} -d www.{domain} --agree-tos
`

Подробнее о certbot можно почитать тут: https://eff-certbot.readthedocs.io/en/stable/using.html

  * Запускаем:
```bash
docker run -d -v ~/certbot/www:/var/www/certbot:rw -v ~/certbot/conf:/etc/letsencrypt:rw --name {certbotName} certbot/certbot:latest certonly --webroot -w /var/www/certbot --force-renewal --email {yourEmail} -d {domain} -d www.{domain} --agree-tos
```
  * Смотри логи
```bash
docker logs {certbotName} --follow
```
Флаг `--follow` для обновления в реальном времени.
  * Если всё хорошо - будет строка `Successfully received certificate.` и пути, куда сохранены сертификаты.
4. Переключаем nginx на работу по ssl
  * Меняем default.conf на:
```conf
server {
    listen 80;
    listen [::]:80;

    server_name {domain} www.{domain}
    server_tokens off;

    location / {
        return 301 https://{domain}$request_uri;
    }
}

server {
    listen 443 default_server ssl http2;
    listen [::]:443 ssl http2;

    server_name {domain} www.{domain};

    ssl_certificate /etc/letsencrypt/live/{domain}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{domain}/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/{domain}/chain.pem;

    ssl_session_cache shared:le_nginx_SSL:10m;
    ssl_session_timeout 1440m;
    ssl_session_tickets off;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";	
    
    location ^~ /path/swagger/ {
        proxy_set_header X-Forwarded-Prefix /path;

        rewrite ^/path/(.*)$ /$1 break;
        proxy_pass http://{serviceName};
    }


    location ^~ /path/api/ {

        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin $http_origin;
            add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
            add_header Access-Control-Allow-Headers 'X-Path, ApiKey, Content-Type';
            add_header Content-Type text/plain;
            add_header Content-Length 0;

            return 204;
        }

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Prefix /path;
        proxy_set_header X-NginX-Proxy true;
        proxy_set_header Host $http_host;

        rewrite ^/path/(.*)$ /$1 break;
        proxy_pass http://{serviceName};
    }

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```
  * Не забудьте везде сменить {domain} на ваш домен.
  * location /api, /swagger приведены для примера.
5. Перезапустить nginx
```bash
docker restart {nginxName}
```
Можно добавить конфиг для блока ip ботов и злоумышленников:
```bash
    #ip blacklist
    include conf.d/blacklist.conf;
```
blacklist.conf:
```bash
deny 43.128.225.120;
...
deny 109.237.97.180;
```
Для применения необходим рестарт когнтейнера.

<a id="CerbotRenew"></a>
### Перевыпуск сертификата 
Осталось настроить автоматический перевыпуск сертификата.

Я буду использовать контейнер certbot и crontab.
  * Удаляем контейнер, который использовался для первого получения сертификата:
```bash
docker rm {cerbot}
```
  * Создаём новый контейнер
```bash
docker container create -v ~/certbot/www:/var/www/certbot:rw -v ~/certbot/conf:/etc/letsencrypt:rw --name {certbotRenewName} certbot/certbot:latest renew --webroot -w /var/www/certbot --dry-run
```
Команда renew запустит обновление сертификата, только если срок его действия подходит к концу.

Для проверки можно добавить параметр --force-renewal
  * Теперь необходимо создать скрипт, который будет вызываться в crontab
```bash
mkdir scripts
cd scripts
vi cert-renew.sh
```
  * Пишем короткий скрипт, который запустит {certbotRenewName} и перезагрузить сертификаты nginx
```bash
# /bin/sh

now=`date +"%d.%m.%Y %T"`
echo [${now}] Updating certificate...
docker start {certbotRenewName}
docker exec {nginxName} nginx -s reload
```
  * Выдаём права:
```bash
chmod 715 cert-renew.sh
```
  * Записываем crontab:
```bash
crontab -u root -e
```
Разумеется, это вместо root можно настроить на другого пользователя. Но в этом случае необходимо убедиться, что у него будет права на скрипт и докер.

Если crontab ещё пуст, то может запросить выбор тектового редактора (позже можно изменить при помощи команды `select-editor`). Мне удобнее использовать /usr/bin/vim.basic.

В файл добавляем вызов скрипта:
```bash
22 3 * * * ~/scripts/cert-renew.sh >>~/scripts/cert-renew.log 2>&1
```
Let's Encrypt рекомендуют запускать обновление два раза в день в случайно выбранную минуту. 
  * Проверяем, на всякий случай
```bash
crontab -u root -l
```
Если мы пишем логи в cert-renew.log, то лучше ещё настроить его периодическую очистку.

Добавим скрипт в папку scripts:
```bash
vi cert-renew-logClear.sh
chmod 715 cert-renew-logClear.sh
```
```bash
# /bin/sh

TRIMpct=70
FILELENGTH=`cat cert-renew.log|wc -l`
LINES2TRIM=`echo ${TRIMpct}*(${FILELENGTH}/100)`
sed -e "1,${LINES2TRIM}d" cert-renew.log >cert-renew.tmp
cat cert-renew.tmp > cert-renew.log
```
И настроим вызов раз в несколько месяцев (время лучше поставить отличное от обновления, чтобы файл не был занят)
```bash
0 15 1 */6 * ~/scripts/cert-renew-logClear.sh >/dev/null 2>&1
```

<a id="Results"></a>
## Итого
В конечном счёте у нас имеется:
1. Контейнер MongoDb, без внешнего доступа (опционально)
2. Том volume для хранения данных mongoDb
3. Контейнер сервиса апи, работающий по сетевому мосту без ssl
4. Сеть мост для связи контейнеров между собой
5. Контейнер Nginx с настроенным SSL, редиректом с http на https
6. Том bind для конфига nginx'a
7. Том bind для хранения сертификатов
8. Том bind для верификации и получения нового сертификата
9. Контейнер certbot для перевыпуска сертификата
10. Задача crontab, инициализующая перевыпуск сертификата

<a id="Notes"></a>
## Заметки
* Отключение/включение подсветки синтаксиса в vim
```bash
:syntax off
:syntax on
```
* Удаление всех контейнеров
```bash
docker rm $(docker ps -aq)
```
* Удаление всех images
```bash
docker rmi $(docker images -q)
```
* Удаление всех неиспользуемых томов
```bash
docker volume prune
```
* Переход в терминал контейнера
```bash
docker exec -t -i {name} /bin/bash
```
* Получить информацию об объекте докера
```bash
docker inspect {name}
```

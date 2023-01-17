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

## Настройка домена <a id="SetDomain"/>
Необходимо прописать IP VPS в A-записи DNS для {domain} и www.{domain}

## <a id="DockerConf"></a> Первоначальная конфигурация, установка Docker.
Подключение из powershell по ssh с паролем (не безопасно, лучше настроить ssh токен):
```bash
ssh -l {ip/domain}
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

## Запуск необходимых контейнеров <a id="RunContainers"/>
Я сознательно не использую docker compose.
Для работы сервиса мне потребуется mongoDB и сам image сервиса.

Для начала поднимем внутренний мост для связи между контейнерами.
```bash
docker network create {bridgeName} --driver bridge
```

### mongoDB <a id="MongoDb"/>
Запуск монги для теста:
```bash
docker run -d -p 27017:27017 --name {mongoName} mongo
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
  -e MONGO_INITDB_ROOT_USERNAME={userName}
  `

  `
  -e MONGO_INITDB_ROOT_PASSWORD={userPassword}
  `
* Чтобы ограничить потребление памяти монгой, можно установить лимит:

  `
  -m 100m
  `
  

Удалить тестовый контейнер можно так:
```bash
docker rm -f {mongoName}
```

Убираем открытый порт 27017 и запускаем монгу:
```bash
docker run -d -m 100m -v {mongoVolumeName}:/data/db -e MONGO_INITDB_ROOT_USERNAME={userName} -e MONGO_INITDB_ROOT_PASSWORD={userPassword} --network {bridgeName} --name {mongoName} mongo
```

### Service API <a id="ServiceAPI"/>
В Dockerfile сервиса убираю EXPOSE 443, т.к. в {bridgeName} трафик будет ходить без ssl.

В Program/Config убираю app.UseHttpsRedirection(), дабы не редиректило.

Публикуем image и запускаем контейнер:
```bash
docker run -d -e MONGODB_CONNSTRING=mongodb://{userName}:{userPassword}@{mongoName} --network {bridgeName} --name {serviceName} {dockerImageName}
```
Можно передать и другие секреты таким образом:

`
-e TELEGRAM_BOTTOKEN={botToken}
`

Из проекта можно легко считать ключи:
```C#
var botToken = Environment.GetEnvironmentVariable("TELEGRAM_BOTTOKEN");
```

Для теста можно было открыть 80 порт tcp:

`
-p 80:80/tcp
`

## Настройка Nginx, SSL <a id="SetNginxSSL"/>
### <a id="Nginx"></a> Nginx
Для nginx'a нам нужно будет расшарить 3 дирректории с использованием bind.
* Bind для конфигурации `/etc/nginx/conf.d/default.conf`
* Bind для получения и проверки сертификата Let's Encrypt `/var/www/certbot`
* Bind для приватного и публичного ключа сертификата `/etc/letsencrypt`

Можно использовать как `-v` так и `--mount`. Однако, `-mount` не создаст дирректорию, если её не существует. А '-v' автоматически определяет bind или volume (если путь - то bind, если строка - то volume.

Запуск nginx с примерами использования `-v` и `-mount`
```bash
docker run -d -p 80:80 -p 443:443 -v ~/certbot/www:/var/www/certbot:rw -v ~/certbot/conf:/etc/letsencrypt:rw --mount type=bind,source=$(pwd)/NginxConf/default.conf,target=/etc/nginx/conf.d/default.conf --network {bridgeName} --name {nginxName} nginx
```
Здесь для mount путь будет относительный, нужно убедиться что вы на верхнем уровне.
### Certbot <a id="Cerbot"/>
Я буду использовать отдельный контейнер с image certbot, но перед получением сертификата необходимо подготовиться.

1. Изменить default.conf для обработки запросов Let's Encrypt:
  * Открываем редактор файла:
```bash
vi ~/NginxConf/default.conf
```
  * Нажимаем INSERT для редактирования
  * Можно отредактировать, но мне удобнее всё удалить и вставить из блокнота. Для удаления всех строк нужно нажать [ESC], ввести `:%d`, нажать [Enter].
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
  * Ждём несколько секунд, и смотрим логи
```bash
docker logs {certbotName}
```
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

    location /api {
        proxy_pass http://{serviceName};
    }

    location /swagger {
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
6. Осталось настроить автоматический перевыпуск сертификата.

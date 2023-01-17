# Шпаргалка для поднятия сервиса на VPS.
_На примере web API ASP .NET Core (.Net 6)_


- [Настройка домена]
- [Первоначальная конфигурация, установка Docker]


## Настройка домена
Необходимо прописать IP VPS в A-записи DNS для {domain} и www.{domain}

## Первоначальная конфигурация, установка Docker.
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

## Запуск необходимых контейнеров
Я сознательно не использую docker compose.
Для работы сервиса мне потребуется mongoDB и сам image сервиса.

Для начала поднимем внутренний мост для связи между контейнерами.
```bash
docker network create {bridgeName} --driver bridge
```

### mongoDB
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

### Service API
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

Для теста можно было открыть 80 порт tcp:

`
-p 80:80/tcp
`

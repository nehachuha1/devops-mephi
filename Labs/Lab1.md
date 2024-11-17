## Установка Docker на Debian

- Установить docker
  ```bash
  sudo apt-get update
  sudo apt-get install ca-certificates curl gnupg
  sudo install -m 0755 -d /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  sudo chmod a+r /etc/apt/keyrings/docker.gpg

  echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  sudo apt-get update

  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

  sudo groupadd docker
  sudo usermod -aG docker $USER
  ```
- Перезайти в shell
- Проверить работоспособность docker комадой (`docker run hello-world`)

## Задания

### 1
- Найти образ nginx на [Docker Hub](https://hub.docker.com/_/nginx) (`docker search nginx`)
- Скачать образ nginx (`docker pull nginx`)
- Проверить, что образ скачался (`docker image ls`)
- Запустить контейнер nginx с пробросом порта (`docker run -d --name nginx nginx`)
- Проверить, что nginx запущен (`docker ps`)
- Проверить доступность nginx на http://localhost. *Почему недоступен?*
- Остановить контейнер (`docker stop nginx`)
- Проверить, что контейнер до сих пор остался висеть в (`docker ps -a`)
- Удалить контейнер (`docker rm nginx`)
- Запустить контейнер nginx с пробросом портов и автоудалением (`docker run -d --rm --name nginx -p 80:80 nginx`)
- Проверить, что nginx запущен (`docker ps`)
- Проверить доступность nginx http://localhost
- Остановить nginx (`docker stop nginx`)
- Убедиться, что контейнер был удален (`docker ps -a`)

### 2
- Запустить контейнер (`docker run -d --rm --name nginx -p 80:80 nginx`)
- Запустить shell в контейнере nginx (`docker exec -it nginx bash`)
- Найти файл сайта (`/usr/share/nginx/html/index.html`)
- Выйти из shell контейнера
- Скопировать файл сайта из контейнера nginx на хост (`docker cp nginx:/usr/share/nginx/html/index.html .`)
- Отредактировать файл index.html на хосте
- Скопировать отредактированный файл index.html в контейнер (`docker cp ./index.html nginx:/usr/share/nginx/html/index.html`)
- Проверить, что сайт поменялся http://localhost

### 3
- Перезапустить контейнер nginx (`docker restart nginx`)
- Убедиться, что изменения на http://localhost остались
- Остановить контейнер nginx (`docker stop nginx`)
- Снова запустить nginx (`docker run -d --rm --name nginx -p 80:80 nginx`)
- Проверить, изменения на http://localhost. *Почему изменения пропали и как можно это исправить?*
- Остановить контейнер nginx (`docker stop nginx`)
- Запустить контейнер nginx с volume (`docker run -d --rm --name nginx -p 80:80 -v .:/usr/share/nginx/html nginx`)
- Изменить файл index.html
- Проверить, что сайт поменялся http://localhost
- Остановить контейнер nginx

### 4
- Написать Dockerfile с nginx и данными
  ```Dockerfile
  FROM nginx
  
  COPY ./index.html /usr/share/nginx/html/index.html
  ```
- Собрать свой образ nginx (`docker build -t mynginx .`)
- Запустить свой образ nginx (`docker run -d --rm --name nginx -p 80:80 mynginx`)
- Проверить, что изменения присутствуют на http://localhost
- Изменить файл index.html
- Проверить http://localhost. *Почему изменения не применились?*
- Остановить контейнер nginx (`docker stop nginx`)

### 5
- Написать docker-compose.yaml для запуска исходного и своего nginx
  ```yaml
  version: "3"
  
  services:
    nginx:
      image: nginx
      container_name: nginx
      ports:
        - "80:80"

    mynginx:
      build: .
      image: mynginx
      container_name: mynginx
      ports:
        - "8080:80"
  ```
- Скачать образы (`docker compose pull`). *Почему возникает ошибка с сервисом mynginx?*
- Собрать образы (`docker compose build`)
- Запустить образы (`docker compose up -d`)
- Убедиться, что образы запущены (`docker ps`, `docker compose ps`)
- Проверить сайты http://localhost и http://localhost:8080

### 6
- Скопировать конфигурацию nginx из образа (`docker cp nginx:/etc/nginx/conf.d/default.conf .`)
- Поменять файл default.conf, добавив туда новый блок после блока `location /`:
  ```conf
  location ^~ /my {
      proxy_pass http://mynginx:80/;
      proxy_redirect off;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Port $server_port;
  }
  ```
- Модифицировать docker-compose.yml, добавив к сервису nginx volume на default.conf
  ```yaml
  version: "3"

  services:
    nginx:
      image: nginx
      container_name: nginx
      volumes:
        - ./default.conf:/etc/nginx/conf.d/default.conf
      ports:
        - "80:80"

    mynginx:
      build: .
      image: mynginx
      container_name: mynginx
      ports:
        - "8080:80"
  ```
- Перезапустить контейнер nginx (`docker compose restart nginx`)
- Убедиться, что образы запущены (`docker ps`, `docker compose ps`)
- Проверить сайты http://localhost, http://localhost:8080, http://localhost/my  


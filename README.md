# Деплой: публикация проекта в Docker на сервере

<ol>
  <li>
    Собрать образы и залить на Docker Hub.
  </li>
  <li>
    Написать такой конфиг для Docker Compose, чтобы он брал образы с Docker Hub.
  </li>
  <li>
    Удалить с сервера проект Taski, оставить только Nginx.
  </li>
  <li>
    Развернуть и запустить Docker на сервере.
  </li>
  <li>
    Загрузить на сервер новый конфиг для Docker Compose и запустить контейнеры.
  </li>
  <li>
    Настроить «внешний» Nginx — тот, что вне контейнера — для работы с приложением в контейнерах.
  </li>
</ol>

## Собираем образы

**Если у вас Mac на Apple Silicon (M1/M2/M3)**

Для сборки образов перейдите в папку taski-docker/ и выполните команды из листинга:
```
cd backend
docker buildx build --platform=linux/amd64 -t username/taski_backend .
cd ../frontend
docker buildx build --platform=linux/amd64 -t username/taski_frontend .
cd ../gateway
docker buildx build --platform=linux/amd64 -t username/taski_gateway .
```

## Загружаем образы на Docker Hub

```
docker push username/taski_backend
docker push username/taski_frontend
docker push username/taski_gateway 
```

## Новый конфиг: docker-compose.production.yml


```
volumes:
  pg_data_production:
  static_volume:

# Всё отличие — заменяем build на image и указываем, какой образ использовать
services:
  db:
    image: postgres:13.10
    env_file: .env
    volumes:
      - pg_data_production:/var/lib/postgresql/data
  backend:
    image: username/taski_backend # Качаем с Docker Hub
    env_file: .env
    volumes:
      - static_volume:/backend_static
  frontend:
    image: username/taski_frontend  # Качаем с Docker Hub
    env_file: .env
    command: cp -r /app/build/. /frontend_static/
    volumes:
      - static_volume:/frontend_static
  gateway:
    image: username/taski_gateway  # Качаем с Docker Hub
    env_file: .env
    volumes:
      - static_volume:/staticfiles/
    ports:
      - 8000:80 
```

Запустите Docker Compose с этой конфигурацией на своём компьютере. Название файла конфигурации надо указать явным образом, ведь оно отличается от дефолтного. Имя файла указывается после ключа -f.

```
docker compose -f docker-compose.production.yml up 
```

Сразу же соберите статику и примените миграции:

```
docker compose -f docker-compose.production.yml exec backend python manage.py collectstatic
docker compose -f docker-compose.production.yml exec backend cp -r /app/collected_static/. /backend_static/static/ 
docker compose -f docker-compose.production.yml exec backend python manage.py migrate
```

Проверьте, что страница http://localhost:8000/api/tasks/ заработала. Если всё хорошо — остановите Docker Compose клавишами Ctrl+C. 

## Чистим сервер: удаляем ненужные файлы и автозапуск Gunicorn

Подключитесь к вашему удалённому серверу:

``` 
# Используйте параметр -i, 
# если файл с SSH-ключом называется не .ssh/id_rsa, а иначе:
ssh -i путь_до_файла_с_SSH_ключом/название_файла_с_SSH_ключом имя_пользователя@ip_адрес_сервера 
```

Остановите Gunicorn и удалите юнит gunicorn, чтобы он больше не перезапускался:

```
sudo systemctl stop gunicorn
sudo rm /etc/systemd/system/gunicorn.service 

```

На удалённом сервере перейдите в домашнюю директорию и удалите содержимое папки taski, но не саму эту папку, также удалите содержимое папки infra_sprint1/

```
cd
rm -r ./taski/*
rm -r ./infra_sprint1/*
```

Очистите кеш npm:
```
npm cache clean --force
```

Очистите кеш APT:

```
sudo apt clean
```
Удалите старые системные логи:

```
sudo journalctl --vacuum-time=1d
```

## Устанавливаем Docker Compose на сервер

```
sudo apt update
sudo apt install curl
curl -fSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
sudo apt install docker-compose-plugin 
```

## Запускаем Docker Compose на сервере

```
scp -i path_to_SSH/SSH_name docker-compose.production.yml \
    username@server_ip:/home/username/taski/docker-compose.production.yml 
```

path_to_SSH — путь к файлу с SSH-ключом;
SSH_name — имя файла с SSH-ключом (без расширения);
username — ваше имя пользователя на сервере;
server_ip — IP вашего сервера.

Скопируйте файл .env на сервер, в директорию taski/.

Для запуска Docker Compose в режиме демона команду docker compose up нужно запустить с флагом -d. Выполните эту команду на сервере в папке taski/:

```
sudo docker compose -f docker-compose.production.yml up -d 
```

Проверьте, что все нужные контейнеры запущены:

```
sudo docker compose -f docker-compose.production.yml ps 
```

Выполните миграции, соберите статические файлы бэкенда и скопируйте их в /backend_static/static/:

```
sudo docker compose -f docker-compose.production.yml exec backend python manage.py migrate
sudo docker compose -f docker-compose.production.yml exec backend python manage.py collectstatic
sudo docker compose -f docker-compose.production.yml exec backend cp -r /app/collected_static/. /backend_static/static/ 
```

## Перенаправляем все запросы в докер

На сервере в редакторе nano откройте конфиг Nginx: sudo nano /etc/nginx/sites-enabled/default. Измените настройки location в секции server.

```
location / {
        root   /var/www/taski;
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:8000;
    }
```

Перезагрузите конфиг Nginx:

```
sudo service nginx reload 
```



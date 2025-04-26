# taski-docker

Откройте терминал, перейдите в директорию backend/ проекта Taski и выполните сборку образа:
```
docker build -t taski_backend . 
```

docker run --name taski_backend_container --rm -p 8000:8000 taski_backend

Выполните миграции в контейнере taski_backend_container:

docker exec taski_backend_container python manage.py migrate 

Перейдите в директорию фронтенда и соберите образ:

docker build -t taski_frontend . 

Запустите контейнер в интерактивном режиме, с ключом -it, и свяжите порт 8000 контейнера с портом 8000 хоста:

docker run --rm -it -p 8000:8000 --name taski_frontend_test taski_frontend

Создайте Docker volume:

docker volume create sqlite_data

Запустите новый контейнер, подключённый к тому же volume:

docker run --name taski_backend_container -p 8000:8000 -v sqlite_data:/data taski_backend

В отдельном окне терминала вновь запустите миграции:

docker exec taski_backend_container python manage.py migrate 

Чтобы убедиться, что данные действительно находятся в постоянном хранилище, остановите и удалите контейнер. В отдельном окне терминала последовательно выполните команды:

docker container stop taski_backend_container
docker container rm taski_backend_container 

Команда удаления тома:

docker volume rm имя_тома

Заново соберите свои образы с нужными именами: откройте в терминале корневую директорию проекта (директорию taski-docker/) и последовательно выполните команды из листинга. Вместо username подставьте логин, с которым вы зарегистрированы на Docker Hub:

# Создать образ (build); 
# присвоить образу имя и тег (-t); 
# Dockerfile взять в указанной директории.
docker build -t alexeyfilichkin/taski_backend:latest backend/
docker build -t alexeyfilichkin/taski_frontend:latest frontend/


Прежде чем загружать образы на Docker Hub, нужно аутентифицировать там докер-демон. Выполните команду аутентификации:

docker login -u alexeyfilichkin

После авторизации можно запушить образы на Docker Hub:

docker push alexeyfilichkin/taski_backend:latest
docker push alexeyfilichkin/taski_frontend:latest 
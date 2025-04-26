# taski-docker

Откройте терминал, перейдите в директорию backend/ проекта Taski и выполните сборку образа:

docker build -t taski_backend . 

docker run --name taski_backend_container --rm -p 8000:8000 taski_backend

Выполните миграции в контейнере taski_backend_container:

docker exec taski_backend_container python manage.py migrate 

Перейдите в директорию фронтенда и соберите образ:

docker build -t taski_frontend . 

Запустите контейнер в интерактивном режиме, с ключом -it, и свяжите порт 8000 контейнера с портом 8000 хоста:

docker run --rm -it -p 8000:8000 --name taski_frontend_test taski_frontend 
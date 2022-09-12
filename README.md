![Build Status](https://github.com/maliarda/yamdb_final/workflows/yamdb_workflow/badge.svg)
# YaMDb API
## Описание

Проект **YaMDb** собирает отзывы (Review) пользователей на произведения (Titles). Произведения делятся на категории: «Книги», «Фильмы», «Музыка». Список категорий (Category) может быть расширен администратором (например, можно добавить категорию «Изобразительное искусство» или «Ювелирка»).

## Пользовательские роли
* **Аноним** — может просматривать описания произведений, читать отзывы и комментарии.
* **Аутентифицированный пользователь** (user) — может, как и Аноним, читать всё, дополнительно он может публиковать отзывы и ставить оценку произведениям (фильмам/книгам/песенкам), может комментировать чужие отзывы; может редактировать и удалять свои отзывы и комментарии. Эта роль присваивается по умолчанию каждому новому пользователю.
* **Модератор** (moderator) — те же права, что и у Аутентифицированного пользователя плюс право удалять любые отзывы и комментарии.
* **Администратор** (admin) — полные права на управление всем контентом проекта. Может создавать и удалять произведения, категории и жанры. Может назначать роли пользователям.
* **Суперюзер Django** — обладет правами администратора (admin)

## Алгоритм регистрации пользователей
1. Пользователь отправляет POST-запрос на добавление нового пользователя с параметрами email и username на эндпоинт /api/v1/auth/signup/.
2. YaMDB отправляет письмо с кодом подтверждения (confirmation_code) на адрес email.
3. Пользователь отправляет POST-запрос с параметрами username и confirmation_code на эндпоинт /api/v1/auth/token/, в ответе на запрос ему приходит token (JWT-токен).
4. При желании пользователь отправляет PATCH-запрос на эндпоинт /api/v1/users/me/ и заполняет поля в своём профайле (описание полей — в документации).

## Как запустить проект:

 _Если у вас не установлены Docker и Docker-compose необходимо воспользоваться официальной [инструкцией](https://docs.docker.com/engine/install/)._

### Клонировать репозиторий и перейти в нем в папку infra в командной строке:

```
git clone https://github.com/Maliarda/yamdb_final.git
```
```
cd yamdb_final/infra
```
### Создать .env файл в директории infra, в котором должны содержаться следующие переменные:
> DB_ENGINE=django.db.backends.postgresql - указываем, что работаем с postgresql

> DB_NAME=postgres - имя базы данных

> POSTGRES_USER=postgres - логин для подключения к базе данных

>POSTGRES_PASSWORD=postgres - пароль для подключения к БД (установите свой)

>DB_HOST=db - название сервиса (контейнера)

> DB_PORT=5432 - порт для подключения к БД

### Создать .env файл в директории api_yamdb для SECRET_KEY
Для генерации нового значения можно использовать команду (из контейнера web, либо иного окружения с установленным python и Django)

```
python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'
```

### Собрать образ при помощи docker-compose

```
docker-compose up -d --build
```

### Применить миграции:
```
docker-compose exec web python manage.py migrate
```
### Собрать статику:
```
docker-compose exec web python manage.py collectstatic --no-input
```
### Добавить данные в базу:
```
docker-compose run web python manage.py loaddata fixtures.json
```
### Создать суперпользователя:
```
docker-compose exec web python manage.py createsuperuser
```

## Установка на удалённом сервере

Необходимо добавить Action secrets в репозитории на GitHub в разделе settings -> Secrets:
* DOCKER_PASSWORD - пароль от DockerHub;
* DOCKER_USERNAME - имя пользователя на DockerHub;
* HOST - ip-адрес сервера;
* SSH_KEY - приватный ssh ключ (публичный должен быть на сервере);
* TELEGRAM_TO - id своего телеграм-аккаунта (можно узнать у @userinfobot, команда /start)
* TELEGRAM_TOKEN - токен бота (получить токен можно у @BotFather, /token, имя бота)

### Проверка работоспособности

Теперь если внести любые изменения в проект и выполнить:
```
git add .
git commit -m "..."
git push
```
Комманда git push является триггером workflow проекта.
При выполнении команды git push запустится набор блоков комманд jobs (см. файл yamdb_workflow.yaml).
Последовательно будут выполнены следующие блоки:
* tests - тестирование проекта на соответствие PEP8 и тестам pytest.
* build_and_push_to_docker_hub - при успешном прохождении тестов собирается образ (image) для docker контейнера 
и отправлятеся в DockerHub
* deploy - после отправки образа на DockerHub начинается деплой проекта на сервере.
Происходит копирование следующих файлов с репозитория на сервер:
  - docker-compose.yaml, необходимый для сборки трех контейнеров:
    + postgres - контейнер базы данных
    + web - контейнер Django приложения + wsgi-сервер gunicorn
    + nginx - веб-сервер
  - nginx/default.conf - файл кофигурации nginx сервера
  - static - папка со статическими файлами проекта
  
  После копировния происходит установка docker и docker-compose на сервере
  и начинается сборка и запуск контейнеров.
* send_message - после сборки и запуска контейнеров происходит отправка сообщения в 
  телеграм об успешном окончании workflow

После выполнения вышеуказанных процедур необходимо установить соединение с сервером:
```
ssh username@server_address
```
Отобразить список работающих контейнеров:
```
sudo docker container ls
```
В списке контейнеров копировать CONTAINER ID контейнера username/yamdb_final_web:latest (username - имя пользователя на DockerHub):
```
CONTAINER ID   IMAGE                            COMMAND                  CREATED          STATUS          PORTS                NAMES
0361a982109d   nginx:1.19.6                     "/docker-entrypoint.…"   50 minutes ago   Up 50 minutes   0.0.0.0:80->80/tcp   yamdb_final_nginx_1
a47ce31d4b7b   username/yamdb_final_web:latest  "/bin/sh -c 'gunicor…"   50 minutes ago   Up 50 minutes                        yamdb_final_web_1
aed19f6751f3   postgres:13.1                    "docker-entrypoint.s…"   50 minutes ago   Up 50 minutes   5432/tcp             yamdb_final_postgres_1
```
Выполнить вход в контейнер:
```
sudo docker exec -it a47ce31d4b7b bash
```
Внутри контейнера выполнить миграции:
```
python manage.py migrate
```
Также можно наполнить базу данных начальными тестовыми данными:
```
python3 manage.py shell
>>> from django.contrib.contenttypes.models import ContentType
>>> ContentType.objects.all().delete()
>>> quit()
python manage.py loaddata fixtures.json
```
Теперь проекту доступна статика. 

Для создания нового суперпользователя можно выполнить команду:
```
$ python manage.py createsuperuser
```
и далее указать: 
```
Email:
Username:
Password:
Password (again):
```

## Примеры запросов:

### Запрос для просмотра категорий анонимным пользователем:
```
    [GET].../api/v1/categories/
```
### Пример ответа:
```
    {
    "count": 3,
    "next": null,
    "previous": null,
    "results": [
        {
            "name": "Книга",
            "slug": "book"
        },
        {
            "name": "Музыка",
            "slug": "music"
        },
        {
            "name": "Фильм",
            "slug": "movie"
        }
    ]
}
```
### Запрос на получение токена от зарегистрированного пользователя:
```
    [POST].../api/v1/auth/token/
{
  "username":"GrumpyCat"
  "confirmation_code":"61o-bb231266379b3908be63"
}
```
### Пример ответа:
```
{
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNjU2MDAxNzkxLCJqdGkiOiIxNjlmOWY1MWE5NzE0MDIyYmI2ZDg2MTFhY2YxMTAyMCIsInVzZXJfaWQiOjEwNn0.C3TPNI7HPl6NWFE0fePPp7G-pF6fEvr0ohtC6me4z4A"
}
```

### Запрос на добавление отзыва пользователем GrumpyCat:
```
    [POST].../api/v1/titles/24/reviews/
{
  "text": "Очень люблю эту книгу. YOU ALWAYS GET BACK TO THE BASICS",
  "score": "10"
}
```
### Пример ответа:
```
{
  "id": 1,
  "author": "GrumpyCat",
  "title": "Generation П",
  "text": "Очень люблю эту книгу. YOU ALWAYS GET BACK TO THE BASICS",
  "pub_date": "2022-06-16T16:35:10.330811Z",
  "score": 10
}   
```


Полная документация доступна по адресу http://localhost/redoc/
## Участники:

[Павел Бойков](https://github.com/pavelboykov). Управление пользователями (Auth и Users): система регистрации и аутентификации, права доступа, работа с токеном, система подтверждения e-mail, поля.

[Мария Пирогова](https://github.com/Maliarda). Категории (Categories), жанры (Genres) и произведения (Titles): модели, view и эндпойнты. README проекта. Docker.

[Михаил Ромашов](https://github.com/Romashovm). Отзывы (Review) и комментарии (Comments): модели и view, эндпойнты. Рейтинги произведений.

![Build Status](https://github.com/maliarda/yamdb_final/workflows/yamdb_workflow/badge.svg)](https://github.com/maliarda/yamdb_final/actions/workflows/yamdb_workflow.yml)
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
git clone https://github.com/Maliarda/infra_sp2.git
```
```
cd infra_sp2/infra
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

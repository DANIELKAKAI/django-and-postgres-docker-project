# Django and Postgresql docker project

## How it works

### Pull a postgresql image with postgis and timescaledb extensions installed
Once the image is pulled we set up password environment variable and run the `start.sql` script to create a new user and database to be used
```
FROM timescale/timescaledb-postgis

ENV PG_DB_PASS password


COPY start.sql /docker-entrypoint-initdb.d/


EXPOSE 5432 
```

### Pull a python image and install django and psycopg2
The psycopg2 module is necessary for database access
```
FROM python:3

ENV PYTHONUNBUFFERED 1

ENV PG_DB_PASS password

RUN mkdir /django-docker

WORKDIR /django-docker

ADD requirements.txt /django-docker/

RUN pip install -r requirements.txt

ADD . /django-docker/

```

### Configuring containers in `docker-compose.yml` file
The django container depends on the postgres container
```yaml
version: '3'

services:
  web:
    container_name: django-app
    build:
      context: .
      dockerfile: Dockerfile-django
    environment:
      - PG_DB_PASS=password
    commands:
      - python manage.py makemigrations
      - python manage.py migrate
      - python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/django-docker
    restart: always
    ports:
      - "8000:8000"
    depends_on:
      - postgresdb
  postgresdb:
    container_name: postgresdb
    build:
      context: .
      dockerfile: Dockerfile-postgres
    restart: always
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - PG_DB_PASS=password

volumes:
  postgres_data:

```
### Django settings
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'my_database',
        'USER':'daniel',
        'HOST':'postgresdb',
        'PASSWORD':os.getenv('PG_DB_PASS',''),
        'PORT':5432
    }
}
```
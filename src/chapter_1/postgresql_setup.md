# PostgreSQL Setup

In this chapter we'll setup PostgreSQL as primary database in our application.

## Setup PostgreSQL Container

Create `db` directory inside our project workspaces
```bash
(venv)$ mkdir db
(venv)$ touch ./db/create.sql
```

Update `create.sql` file with following content
```sql
CREATE DATABASE todos_production;
CREATE DATABASE todos_development;
CREATE DATABASE todos_testing;

```

Create `Dockerfile` inside `db` directory with following content

```
FROM postgres:14.6-alpine

COPY create.sql /docker-entrypoint-initdb.d
```

Update `docker-compose.yml` to

> This is for educational purpose only, consider to hash your credential before commit your code.

```yaml
version: '3.8'

services:
  postgres:
    build: ./db
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password

  todos:
    build: .
    command: python serve.py
    volumes:
      - .:/usr/src/app/
    ports:
      - 5000:5000
    env_file:
      - ./.env.dev
    depends_on:
      postgres:
        condition: service_started

volumes:
  postgres_data:

```

After setup database in docker container, we can add required library to our app for this project.
Update our `requirements.txt` to following content

```txt
Flask==2.2.2
SQLAlchemy==1.4.45
Flask-SQLAlchemy==3.0.2
Flask-Migrate==4.0.0
psycopg2-binary==2.9.5

```

Before build our image, update main `Dockerfile` to following content

```Dockerfile
FROM python:3.8-slim

# Set environment varibles
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# Install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .

RUN pip install -r requirements.txt

COPY . .

```

Then we can rebuild our docker image by invoke command
```bash
$ docker compose down
$ docker compose build
$ docker compose up -d
```

To check wether our database created or not, we can invoke `psql` from our `postgres` container
```bash
$ docker compose exec postgres psql -U postgres
```

When in `psql` interface we can invoke `\l` to list database in the container
```
postgres=# \l
                                     List of databases
       Name        |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-------------------+----------+----------+------------+------------+-----------------------
 postgres          | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0         | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                   |          |          |            |            | postgres=CTc/postgres
 template1         | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                   |          |          |            |            | postgres=CTc/postgres
 todos_development | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 todos_production  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 todos_testing     | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
(6 rows)
```

Next we can setup database connection on our application config

> This is for educational purpose only, consider to hash your credential before commit your code.

```python
import os
from functools import lru_cache


class Config:
    DEBUG = False
    SQLALCHEMY_TRACK_MODIFICATIONS = False


class ProductionConfig(Config):
    SQLALCHEMY_DATABASE_URI = "postgresql://postgres:password@postgres:5432/todos_production"


class DevelopmentConfig(Config):
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = "postgresql://postgres:password@postgres:5432/todos_development"


class TestingConfig(Config):
    TESTING=True
    SQLALCHEMY_DATABASE_URI = "postgresql://postgres:password@postgres:5432/todos_testing"


@lru_cache
def get_config():
    config = {
        "production": ProductionConfig,
        "development": DevelopmentConfig,
        "testing": TestingConfig,
    }
    env = os.getenv("APP_ENV", "development")
    return config.get(env, DevelopmentConfig)

```

Then update our `factory.py` script to
```python
from typing import List, Type, Union

from flask import Blueprint, Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

from app.config import Config
from app.http.route import Route

db = SQLAlchemy()
migrate = Migrate()

def create_app(
    app_name: str, config: Type[Config], routes: List[Union[Route, Blueprint]]
):
    app = Flask(app_name)
    app.config.from_object(config)

    db.init_app(app)
    migrate.init_app(app, db)

    for route in routes:
        if isinstance(route, Blueprint):
            app.register_blueprint(route)
        else:
            app.add_url_rule(
                route.url_rule, view_func=route.view_func(), methods=route.methods()
            )

    return app

```

Then run migration init to generate migration related scripts

```bash
(venv)$ flask db init
```

This will create directory `migrations` to store our next model migrations.

Your final project structure should be look like this

```
flask-todo
├── app
│   ├── config.py
│   ├── factory.py
│   ├── http
│   │   ├── __init__.py
│   │   └── route.py
│   ├── __init__.py
│   ├── routes
│   │   ├── api.py
│   │   └── __init__.py
│   └── views
│       ├── __init__.py
│       └── main_api.py
├── db
│   ├── create.sql
│   └── Dockerfile
├── docker-compose.yml
├── Dockerfile
├── .dockerignore
├── .env.dev
├── .gitignore
├── migrations
│   ├── alembic.ini
│   ├── env.py
│   ├── README
│   ├── script.py.mako
│   └── versions
├── requirements.txt
└── serve.py

7 directories, 22 files
```

We can commit our works before continuing to next chapter.

```bash
(venv)$ git add .
(venv)$ git commit -m "PostgreSQL Connection Setup"
```

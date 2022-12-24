# Docker Setup

> This chapter require [docker](https://docs.docker.com/get-docker/) and docker compose installed

Let's put our application inside docker container

## Setup Application Container

Create `Dockerfile` in our project directory

```Dockerfile
FROM python:3.8-alpine

# Set environment varibles
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

WORKDIR /usr/src/app

# Install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .

RUN pip install -r requirements.txt

COPY . .

```

Create `.dockerignore` file to avoid unnecessarily sending large or sensitive files and directories to the daemon and potentially adding them to images.

```dockerignore
__pycache__/
venv/
Dockerfile
env
.env

```

Create `docker-compose.yml` file to manage our container via docker compose
```yaml
version: '3.8'

services:
  todos:
    build: .
    command: python serve.py
    volumes:
      - .:/usr/src/app/
    ports:
      - 5000:5000
    env_file:
      - ./.env.dev

```

Create file `.env.dev` to add env based configuration
```env
APP_ENV=development
```

Before build our image, we should add slight changes to our `serve.py`

```python
from app import app

if __name__ == "__main__":
    app.run(host="0.0.0.0")

```

This will tell our application to use address `0.0.0.0` instead of `127.0.0.1`.

Next we can build our docker with command
```bash
$ docker compose build
```

After finish the build, we can start our services with
```bash
$ docker compose up -d
```

Navigating to [http://localhost:5000](http://localhost:5000) should return same response as before
```json
{
  "message": "Application running."
}
```

We can check our container logs with command `docker compose logs {container_name} -f`
```bash
$ docker compose logs todos -f
```

To shutdown our container we can invoke `docker compose down -v`

Your final project structure should look like this
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
├── docker-compose.yml
├── Dockerfile
├── .dockerignore
├── .env.dev
├── .gitignore
├── requirements.txt
└── serve.py

4 directories, 16 files

```

We can commit our works before continuing to next chapter.

```bash
(venv)$ git add .
(venv)$ git commit -m "Docker Setup"
```
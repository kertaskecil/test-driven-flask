# Create ToDo API Endpoint

In this chapter, we will create endpoint `/api/todos` to create `Task` based on user needs.

## Write the Test
Writing test for Flask application actually like writing function, but it have a few rules

- Test file name should starts with `test_*.py` or end with `*_test.py`, in this tutorial we will use latest pattern `*_test.py`
- Test method or function should start with `test_`
- In this tutorial we will cover using python standard library for unit testing `unittest`

Before create test case for our project, we need to define our base class that hold our test app. Let's create `tests/base.py`

```python
import unittest
from app.factory import create_app
from app.config import TestingConfig
from app.routes.api import api_routes


class BaseAPITestCase(unittest.TestCase):
    def setUp(self) -> None:
        self.app = create_app("test-app", config=TestingConfig, routes=api_routes)
        self.client = self.app.test_client()
        self.ctx = self.app.app_context()
        self.ctx.push()

    def tearDown(self) -> None:
        self.ctx.pop()

```
This class will hold our app with `TestingConfig`, which connected to our testing specific database `todos_testing`.

Let's write our first test by create new file `create_task_api_test.py` to `tests/features/tasks` directory.

```python
# tests/features/task/create_task_api_test.py

from http import HTTPStatus
from tests.base import BaseAPITestCase
from app.tasks.models.task import Task
from app.factory import db


class CreateTaskAPITest(BaseAPITestCase):
    def test_create_task(self):
        endpoint = "/api/todos"
        json_data = {
            "name": "Fix handrawer",
        }

        resp = self.client.post(
            endpoint,
            json=json_data
        )
        self.assertEqual(resp.status_code, HTTPStatus.CREATED)

        tasks = Task.query.all()
        self.assertEqual(1, len(tasks))

        Task.query.delete()
        db.session.commit()

```

Ensure your container is up and running by invoke command.

```bash
$ docker compose up -d
```
Then run the test with python `unittest` module.
```bash
$ docker compose exec todos python -m unittest tests/features/task/create_task_api.py
```

It will return failure message
```
F
======================================================================
FAIL: test_create_task (tests.features.task.create_task_api.CreateTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/create_task_api.py", line 20, in test_create_task
    self.assertEqual(resp.status_code, HTTPStatus.CREATED)
AssertionError: 404 != <HTTPStatus.CREATED: 201>

----------------------------------------------------------------------
Ran 1 test in 0.006s

FAILED (failures=1)
```

This occured because we not define our `Task` resource yet. To define `Task` resource, first we need to create table to store the data by creating new migration, first we need to define our models.

First we need to create test againts our `Task` model, create file `tests/unit/task/task_model_test.py`

```python
from app.factory import db
from app.tasks.models.task import Task
from tests.base import BaseAPITestCase


class TaskModelTest(BaseAPITestCase):
    def test_create_task(self):
        task = Task(name="Finish Homework")
        db.session.add(task)
        db.session.commit()

        tasks = Task.query.all()
        assert len(tasks) == 1

        db.session.delete(task)
        db.session.commit()

```

Then run the test
```bash
$ docker compose exec todos python -m unittest tests/unit/task/task_model_test.py
```

It will return failure message
```
E
======================================================================
ERROR: task_model_test (unittest.loader._FailedTest)
----------------------------------------------------------------------
ImportError: Failed to import test module: task_model_test
Traceback (most recent call last):
  File "/usr/local/lib/python3.8/unittest/loader.py", line 154, in loadTestsFromName
    module = __import__(module_name)
  File "/usr/src/app/tests/unit/task/task_model_test.py", line 2, in <module>
    from app.modules.tasks.models import Task
ImportError: cannot import name 'Task' from 'app.modules.tasks.models' (/usr/src/app/app/modules/tasks/models.py)


----------------------------------------------------------------------
Ran 1 test in 0.000s

FAILED (errors=1)
```

Create `Task` model to file `app/tasks/models/task.py`

```python
# app/tasks/models/task.py

from app.factory import db


class Task(db.Model): # type: ignore
    __tablename__ = "tasks"

    id = db.Column(db.BigInteger, primary_key=True)
    name = db.Column(db.String(255))
    completed = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=db.func.current_timestamp())
    updated_at = db.Column(
        db.DateTime,
        default=db.func.current_timestamp(),
        onupdate=db.func.current_timestamp(),
    )
```

Apparently our Models is not known to `alembic` migrations until it loaded to our app via routing registration, we need to create empty Handler for our routing.
Create file `app/tasks/views/create_task_api.py` contains basic handler view.

```python
# app/tasks/views/create_task_api.py

from flask import jsonify
from flask.typing import ResponseReturnValue
from flask.views import View

from app.tasks.models.task import Task

class CreateTaskAPI(View):
    def dispatch_request(self) -> ResponseReturnValue:
        return jsonify({})

```

Then register the handler to our `app/routes/api.py`

```python
# app/routes/api.py

from app.http.route import Route
from app.tasks.views.create_task_api import CreateTaskAPI
from app.views.main_api import MainView

api_routes = [
    Route.get("/", MainView),
    Route.group("/api", routes=[
        Route.post("/todos", view=CreateTaskAPI)
    ]),
]
```
In this routing file, we group our `/todos` endpoint to `/api` group.

Generate db migration by running command
```bash
$ docker compose exec todos flask db migrate -m "create tasks table"
```

This will create new generated file on `migrations/versions/{hash_number}_{message}.py`

Then we can run our migration againts `testing` environment with command
```bash
$ docker compose exec -e APP_ENV=testing todos flask db upgrade
```

When we run our model test againts newly created table, it will return success
```bash
$ docker compose exec todos python -m unittest tests/unit/task/task_model_test.py
```

It will return message
```
.
----------------------------------------------------------------------
Ran 1 test in 0.027s

OK
```

Next try re-running our feature test `tests/features/task/create_task_api_test.py`
```bash
$ docker compose exec todos python -m unittest tests/features/task/create_task_api_test.py
```
It does not return HTTP Not Found Error (404) anymore but still fail our test

```
F
======================================================================
FAIL: test_create_task (tests.features.task.create_task_api_test.CreateTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/create_task_api_test.py", line 16, in test_create_task
    self.assertEqual(resp.status_code, HTTPStatus.CREATED)
AssertionError: 200 != <HTTPStatus.CREATED: 201>

----------------------------------------------------------------------
Ran 1 test in 0.007s

FAILED (failures=1)
```

Let's make it green by implementing code at `app/tasks/views/create_task_api.py`
```python
# app/tasks/views/create_task_api.py

from http import HTTPStatus

from flask import jsonify, request
from flask.typing import ResponseReturnValue
from flask.views import View

from app.factory import db
from app.tasks.models.task import Task


class CreateTaskAPI(View):
    def dispatch_request(self) -> ResponseReturnValue:
        json_data = request.get_json()

        task = Task(**json_data)

        db.session.add(task)
        db.session.commit()
        resp = {
            "msg": "Created"
        }
        return jsonify(resp), HTTPStatus.CREATED

```

Re-run the test `tests/features/task/create_task_api_test.py`
```bash
$ docker compose exec todos python -m unittest tests/features/task/create_task_api_test.py
```

It will pass
```
.
----------------------------------------------------------------------
Ran 1 test in 0.031s

OK
```

But this test only covers happy path only. We need to add tests for following scenario

- Empty json paylod
- Wrong json attributes

Before adding new tests, lets create helper to help invoke our test. Create file `bin/test` on our application directory.
```
#!/usr/bin/env bash

docker compose exec todos python -m unittest $@
```

Make file executable with
```bash
$ chmod +x bin/test
```

Now we can invoke our test with `bin/test unit_test.py`.

Let's add test for empty json payload to our test case
```python
# tests/features/task/create_task_api_test.py
    ...
    def test_create_task_when_payload_empty(self):
        endpoint = "/api/todos"
        json_data = {}

        resp = self.client.post(
            endpoint,
            json=json_data
        )
        self.assertEqual(resp.status_code, HTTPStatus.BAD_REQUEST)

        Task.query.delete()
        db.session.commit()
```

Re-run test with
```bash
$ bin/test tests/features/task/create_task_api_test.py
```

It will raise error `NotNullViolation`, because attribute name is not allowed to be null.
```
.E
======================================================================
ERROR: test_create_task_when_payload_empty (tests.features.task.create_task_api_test.CreateTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/local/lib/python3.8/site-packages/sqlalchemy/engine/base.py", line 1900, in _execute_context
    self.dialect.do_execute(
  File "/usr/local/lib/python3.8/site-packages/sqlalchemy/engine/default.py", line 736, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.NotNullViolation: null value in column "name" of relation "tasks" violates not-null constraint
```

To validate user request schema, let's add `marshmallow` package to our application code base by update `requirements.txt`.
```txt
Flask==2.2.2
SQLAlchemy==1.4.45
Flask-SQLAlchemy==3.0.2
Flask-Migrate==4.0.0
psycopg2==2.9.5
marshmallow==3.19.0
marshmallow-sqlalchemy==0.28.1
flask-marshmallow==0.14.0
```

Then rebuild our container with `docker compose build` command, then update our running container with `docker compose up -d`.
After add `flask-marshmallow` package, then register to our application factory.

```python
# app/factory.py

from typing import List, Type, Union

from flask import Blueprint, Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_marshmallow import Marshmallow

from app.config import Config
from app.http.route import Route

db = SQLAlchemy()
migrate = Migrate()
ma = Marshmallow()

def create_app(
    app_name: str, config: Type[Config], routes: List[Union[Route, Blueprint]]
):
    app = Flask(app_name)
    app.config.from_object(config)

    db.init_app(app)
    ma.init_app(app)
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

Then create our `TaskSchema` on `app/tasks/schemas/task_schema.py`
```python
# app/tasks/schemas/task_schema.py

from marshmallow import fields, Schema


class TaskSchema(Schema):
    id = fields.Integer(dump_only=True)
    name = fields.String(required=True)
    completed = fields.Boolean()
    created_at = fields.DateTime(dump_only=True)
    updated_at = fields.DateTime(dump_only=True)

```

Next implement validation on our view at `app/tasks/views/create_task_api.py`
```python
# app/tasks/views/create_task_api.py

from http import HTTPStatus

from flask import jsonify, make_response, request
from flask.typing import ResponseReturnValue
from flask.views import View

from app.factory import db
from app.tasks.models.task import Task
from app.tasks.schemas.task_schema import TaskSchema


class CreateTaskAPI(View):
    def dispatch_request(self) -> ResponseReturnValue:
        task_schema = TaskSchema()
        json_data = request.get_json() or dict()
        errors = task_schema.validate(json_data)
        if errors:
            return make_response(
                jsonify(errors=str(errors)),
                HTTPStatus.BAD_REQUEST
            )

        task = Task(**json_data)

        db.session.add(task)
        db.session.commit()
        resp = {
            "data": task_schema.dump(task)
        }
        return jsonify(resp), HTTPStatus.CREATED

```

Re-run test with
```bash
$ bin/test tests/features/task/create_task_api_test.py
```

Test passing
```
..
----------------------------------------------------------------------
Ran 2 tests in 0.041s

OK
```

Next let's create test by passing wrong attributes
```python
    ...
    def test_create_task_when_payload_has_unknown_attribute(self):
        endpoint = "/api/todos"
        json_data = {
            "name": "Fix new car",
            "completed": False,
            "other": "value",
        }

        resp = self.client.post(
            endpoint,
            json=json_data
        )
        self.assertEqual(resp.status_code, HTTPStatus.BAD_REQUEST)

        Task.query.delete()
        db.session.commit()
```

Re-run test with
```bash
$ bin/test tests/features/task/create_task_api_test.py
```

Test passing
```
...
----------------------------------------------------------------------
Ran 3 tests in 0.052s

OK
```

After our scenario has been implemented, we can run all our test we've created so far by running command
```bash
$ bin/test discover -s tests -p '*_test.py'
```

Ensure all test are passed
```
....
----------------------------------------------------------------------
Ran 4 tests in 0.064s

OK
```

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
│   ├── tasks
│   │   ├── __init__.py
│   │   ├── models
│   │   │   ├── __init__.py
│   │   │   └── task.py
│   │   ├── schemas
│   │   │   ├── __init__.py
│   │   │   └── task_schema.py
│   │   └── views
│   │       ├── create_task_api.py
│   │       └── __init__.py
│   └── views
│       ├── __init__.py
│       └── main_api.py
├── bin
│   └── test
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
│       └── 213adaf34061_create_tasks_tables.py
├── requirements.txt
├── serve.py
└── tests
    ├── base.py
    ├── features
    │   ├── __init__.py
    │   └── task
    │       ├── create_task_api_test.py
    │       └── __init__.py
    ├── __init__.py
    └── unit
        ├── __init__.py
        └── task
            ├── __init__.py
            └── task_model_test.py

17 directories, 39 files
```

We can commit our works before continuing to next chapter.

```bash
(venv)$ git add .
(venv)$ git commit -m "Create CreateTaskAPI endpoint"
```
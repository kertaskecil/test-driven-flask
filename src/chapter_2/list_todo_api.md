# List ToDo API

In this chapter we will create endpoint `/api/todos` to list all created `Task` in our database.

## Writing Tests

First let's create test at `tests/features/task/list_task_api_test.py`
```python
from http import HTTPStatus
from tests.base import BaseAPITestCase
from app.tasks.models.task import Task
from app.factory import db


class ListTaskAPITest(BaseAPITestCase):
    def test_list_task_api(self):
        endpoint = "/api/todos"
        task_1 = Task(name="Finish Homework")
        task_2 = Task(name="Clean up closet")
        db.session.add_all([task_1, task_2])
        db.session.commit()

        resp = self.client.get(endpoint)

        self.assertEqual(resp.status_code, HTTPStatus.OK)

        resp_json = resp.json or dict()
        data = resp_json["data"]

        self.assertEqual(data[0]["name"], "Finish Homework")
        self.assertEqual(data[1]["name"], "Clean up closet")

        Task.query.delete()
        db.session.commit()
```

Invoke test by running command
```bash
$ bin/test tests/features/task/list_task_api_test.py
```

It will return error
```
F
======================================================================
FAIL: test_list_task_api (tests.features.task.list_task_api_test.ListTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/list_task_api_test.py", line 17, in test_list_task_api
    self.assertEqual(resp.status_code, HTTPStatus.OK)
AssertionError: 405 != <HTTPStatus.OK: 200>

----------------------------------------------------------------------
Ran 1 test in 0.019s

FAILED (failures=1)
```

It raise error 405 (HTTP Method Not Allowed) since we not implement our routes yet.
Let's implement our routes at file `app/tasks/views/list_task_api.py`

```python
from http import HTTPStatus

from flask import jsonify
from flask.typing import ResponseReturnValue
from flask.views import View

from app.factory import db
from app.tasks.models.task import Task
from app.tasks.schemas.task_schema import TaskSchema


class ListTaskAPI(View):
    def dispatch_request(self) -> ResponseReturnValue:
        tasks_schema = TaskSchema(many=True)
        tasks = Task.query.all()

        resp = {
            "data": tasks_schema.dump(tasks)
        }
        return jsonify(resp), HTTPStatus.OK
```

Then register it `app/routes/api.py`
```python
from app.http.route import Route
from app.tasks.views.list_task_api import ListTaskAPI
from app.tasks.views.create_task_api import CreateTaskAPI
from app.views.main_api import MainView

api_routes = [
    Route.get("/", MainView),
    Route.group("/api", routes=[
        Route.post("/todos", view=CreateTaskAPI),
        Route.get("/todos", view=ListTaskAPI),
    ]),
]
```

Let's re run the test
```bash
$ bin/test tests/features/task/list_task_api_test.py
```

It should passed
```
.
----------------------------------------------------------------------
Ran 1 test in 0.031s

OK
```

Re-run all test to ensure our new feature not breaking existing feature
```bash
$ bin/test discover -s tests -p '*_test.py'
```

It should pass all tests
```
.....
----------------------------------------------------------------------
Ran 5 tests in 0.084s

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
│   │       ├── __init__.py
│   │       └── list_task_api.py
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
    │       ├── __init__.py
    │       └── list_task_api_test.py
    ├── __init__.py
    └── unit
        ├── __init__.py
        └── task
            ├── __init__.py
            └── task_model_test.py

17 directories, 41 files
```

We can commit our works before continuing to next chapter.

```bash
(venv)$ git add .
(venv)$ git commit -m "Create ListTaskAPI endpoint"
```
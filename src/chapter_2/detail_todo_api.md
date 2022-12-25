# Detail ToDo API

In this chapter we will create endpoint `/api/todos/{:id}` to get detail about specific `Task`

## Writing Tests

Write test at `tests/features/task/detail_task_api_test.py`
```python
from http import HTTPStatus
from tests.base import BaseAPITestCase
from app.tasks.models.task import Task
from app.factory import db


class DetailTaskAPITest(BaseAPITestCase):
    def test_detail_task(self):
        endpoint = "/api/todos"
        task = Task(name="Buy groceries")
        db.session.add(task)
        db.session.commit()

        db.session.refresh(task)

        resp = self.client.get(f"{endpoint}/{task.id}")
        self.assertEqual(resp.status_code, HTTPStatus.OK)

        resp_json = resp.json or dict()
        data = resp_json["data"]
        self.assertEqual(data["name"], "Buy groceries")

        Task.query.delete()
        db.session.commit()
```

It will raise error HTTP_NOT_FOUND (404), since we not implement our code
```
F
======================================================================
FAIL: test_create_task (tests.features.task.detail_task_api_test.CreateTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/detail_task_api_test.py", line 17, in test_create_task
    self.assertEqual(resp.status_code, HTTPStatus.OK)
AssertionError: 404 != <HTTPStatus.OK: 200>

----------------------------------------------------------------------
Ran 1 test in 0.029s

FAILED (failures=1)
```

Let's implement our code at `app/tasks/views/detail_task_api.py`
```python
from http import HTTPStatus

from flask import jsonify
from flask.typing import ResponseReturnValue
from flask.views import View

from app.tasks.models.task import Task
from app.tasks.schemas.task_schema import TaskSchema


class DetailTaskAPI(View):
    def dispatch_request(self, task_id) -> ResponseReturnValue:
        tasks_schema = TaskSchema()
        task = Task.query.filter(
            Task.id == task_id
        ).first()

        resp = {
            "data": tasks_schema.dump(task)
        }
        return jsonify(resp), HTTPStatus.OK
```

And update our api routes to
```python
from app.http.route import Route
from app.tasks.views.detail_task_api import DetailTaskAPI
from app.tasks.views.list_task_api import ListTaskAPI
from app.tasks.views.create_task_api import CreateTaskAPI
from app.views.main_api import MainView

api_routes = [
    Route.get("/", MainView),
    Route.group("/api", routes=[
        Route.post("/todos", view=CreateTaskAPI),
        Route.get("/todos", view=ListTaskAPI),
        Route.get("/todos/<task_id>", view=DetailTaskAPI),
    ]),
]
```

Re-run our test file
```bash
$ bin/test tests/features/task/detail_task_api_test.py
```

It should pass
```
.
----------------------------------------------------------------------
Ran 1 test in 0.033s

OK
```

Let's add test to cover when `id` passed not available
```python
    ...
    def test_detail_task_when_id_not_exists(self):
        endpoint = "/api/todos"
        resp = self.client.get(f"{endpoint}/123")
        self.assertEqual(resp.status_code, HTTPStatus.NOT_FOUND)
```

Re-run our test file
```bash
$ bin/test tests/features/task/detail_task_api_test.py
```

It should raise error
```
.F
======================================================================
FAIL: test_detail_task_when_id_not_exists (tests.features.task.detail_task_api_test.DetailTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/detail_task_api_test.py", line 29, in test_detail_task_when_id_not_exists
    self.assertEqual(resp.status_code, HTTPStatus.NOT_FOUND)
AssertionError: 200 != <HTTPStatus.NOT_FOUND: 404>

----------------------------------------------------------------------
Ran 2 tests in 0.046s

FAILED (failures=1)
```

Let's change our view to
```python
from http import HTTPStatus

from flask import jsonify
from flask.typing import ResponseReturnValue
from flask.views import View

from app.tasks.models.task import Task
from app.tasks.schemas.task_schema import TaskSchema


class DetailTaskAPI(View):
    def dispatch_request(self, task_id) -> ResponseReturnValue:
        tasks_schema = TaskSchema()
        task = Task.query.filter(
            Task.id == task_id
        ).first_or_404()

        resp = {
            "data": tasks_schema.dump(task)
        }
        return jsonify(resp), HTTPStatus.OK
```

Re-run our test file
```bash
$ bin/test tests/features/task/detail_task_api_test.py
```

It should be pass
```
..
----------------------------------------------------------------------
Ran 2 tests in 0.047s

OK
```

Let's re-run all of our tests
```bash
$ bin/test discover -s tests -p '*_test.py'
```

It should pass
```
.......
----------------------------------------------------------------------
Ran 7 tests in 0.123s

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
│   │       ├── detail_task_api.py
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
    │       ├── detail_task_api_test.py
    │       ├── __init__.py
    │       └── list_task_api_test.py
    ├── __init__.py
    └── unit
        ├── __init__.py
        └── task
            ├── __init__.py
            └── task_model_test.py

17 directories, 43 files
```

We can commit our works before continuing to next chapter.

```bash
(venv)$ git add .
(venv)$ git commit -m "Create DetailTaskAPI endpoint"
```

# Update ToDo API

In this chapter we will create endpoint `/api/todos/{:id}` to update our `Task` model

## Writing Test

Write test at `tests/features/task/update_task_api_test.py`
```python
from http import HTTPStatus
from tests.base import BaseAPITestCase
from app.tasks.models.task import Task
from app.factory import db


class UpdateTaskAPITest(BaseAPITestCase):
    def test_update_task(self):
        endpoint = "/api/todos"
        task = Task(name="Buy groceries")
        db.session.add(task)
        db.session.commit()

        db.session.refresh(task)

        json_data = {
            "name": "Buy milk",
            "completed": True,
        }
        resp = self.client.post(
            f"{endpoint}/{task.id}",
            json=json_data
        )
        self.assertEqual(resp.status_code, HTTPStatus.OK)

        resp_json = resp.json or dict()
        data = resp_json["data"]
        self.assertEqual(data["name"], "Buy milk")
        self.assertEqual(data["completed"], True)

        Task.query.delete()
        db.session.commit()
```

Run our test
```bash
$ bin/test tests/features/task/update_task_api_test.py
```

It will error
```
F
======================================================================
FAIL: test_update_task (tests.features.task.update_task_api_test.UpdateTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/update_task_api_test.py", line 24, in test_update_task
    self.assertEqual(resp.status_code, HTTPStatus.OK)
AssertionError: 405 != <HTTPStatus.OK: 200>

----------------------------------------------------------------------
Ran 1 test in 0.032s

FAILED (failures=1)
```

Let's implement our view at `app/tasks/views/update_task_api.py`
```python
from http import HTTPStatus

from flask import jsonify, make_response, request
from flask.typing import ResponseReturnValue
from flask.views import View

from app.factory import db
from app.tasks.models.task import Task
from app.tasks.schemas.task_schema import TaskSchema


class UpdateTaskAPI(View):
    def dispatch_request(self, task_id) -> ResponseReturnValue:
        task_schema = TaskSchema()
        json_data = request.get_json() or dict()

        task = Task.query.filter(
            Task.id == task_id
        ).first()

        task.query.update(json_data)
        db.session.commit()

        resp = {
            "data": task_schema.dump(task)
        }
        return jsonify(resp), HTTPStatus.OK
```

Update our routes to
```python
from app.http.route import Route
from app.tasks.views.detail_task_api import DetailTaskAPI
from app.tasks.views.list_task_api import ListTaskAPI
from app.tasks.views.create_task_api import CreateTaskAPI
from app.tasks.views.update_task_api import UpdateTaskAPI
from app.views.main_api import MainView

api_routes = [
    Route.get("/", MainView),
    Route.group("/api", routes=[
        Route.post("/todos", view=CreateTaskAPI),
        Route.get("/todos", view=ListTaskAPI),
        Route.get("/todos/<task_id>", view=DetailTaskAPI),
        Route.put("/todos/<task_id>", view=UpdateTaskAPI),
    ]),
]
```

Re-run the test
```bash
$ bin/test tests/features/task/update_task_api_test.py
```

It should pass
```
.
----------------------------------------------------------------------
Ran 1 test in 0.041s

OK
```

Let's add test when updated `Task` does not exists
```python
    ...
    def test_update_task_when_id_not_exists(self):
    endpoint = "/api/todos"
    json_data = {
        "name": "Buy milk tea",
    }
    resp = self.client.put(f"{endpoint}/123", json=json_data)
    self.assertEqual(resp.status_code, HTTPStatus.NOT_FOUND)
```

Re-run the test
```bash
$ bin/test tests/features/task/update_task_api_test.py
```

It should fail
```
.E
======================================================================
ERROR: test_update_task_when_id_not_exists (tests.features.task.update_task_api_test.UpdateTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/update_task_api_test.py", line 39, in test_update_task_when_id_not_exists
    resp = self.client.put(f"{endpoint}/123", json=json_data)
  File "/usr/local/lib/python3.8/site-packages/werkzeug/test.py", line 1150, in put
    return self.open(*args, **kw)
  File "/usr/local/lib/python3.8/site-packages/flask/testing.py", line 223, in open
    response = super().open(
  File "/usr/local/lib/python3.8/site-packages/werkzeug/test.py", line 1094, in open
    response = self.run_wsgi_app(request.environ, buffered=buffered)
  File "/usr/local/lib/python3.8/site-packages/werkzeug/test.py", line 961, in run_wsgi_app
    rv = run_wsgi_app(self.application, environ, buffered=buffered)
  File "/usr/local/lib/python3.8/site-packages/werkzeug/test.py", line 1242, in run_wsgi_app
    app_rv = app(environ, start_response)
  File "/usr/local/lib/python3.8/site-packages/flask/app.py", line 2548, in __call__
    return self.wsgi_app(environ, start_response)
  File "/usr/local/lib/python3.8/site-packages/flask/app.py", line 2528, in wsgi_app
    response = self.handle_exception(e)
  File "/usr/local/lib/python3.8/site-packages/flask/app.py", line 2525, in wsgi_app
    response = self.full_dispatch_request()
  File "/usr/local/lib/python3.8/site-packages/flask/app.py", line 1822, in full_dispatch_request
    rv = self.handle_user_exception(e)
  File "/usr/local/lib/python3.8/site-packages/flask/app.py", line 1820, in full_dispatch_request
    rv = self.dispatch_request()
  File "/usr/local/lib/python3.8/site-packages/flask/app.py", line 1796, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)
  File "/usr/local/lib/python3.8/site-packages/flask/views.py", line 107, in view
    return current_app.ensure_sync(self.dispatch_request)(**kwargs)
  File "/usr/src/app/app/tasks/views/update_task_api.py", line 28, in dispatch_request
    task.query.update(json_data)
AttributeError: 'NoneType' object has no attribute 'query'

----------------------------------------------------------------------
Ran 2 tests in 0.057s

FAILED (errors=1)
```

Fix our implementation to
```python
from http import HTTPStatus

from flask import jsonify, make_response, request
from flask.typing import ResponseReturnValue
from flask.views import View

from app.factory import db
from app.tasks.models.task import Task
from app.tasks.schemas.task_schema import TaskSchema


class UpdateTaskAPI(View):
    def dispatch_request(self, task_id) -> ResponseReturnValue:
        task_schema = TaskSchema()
        json_data = request.get_json() or dict()

        task = Task.query.filter(
            Task.id == task_id
        ).first_or_404()

        task.query.update(json_data)
        db.session.commit()

        resp = {
            "data": task_schema.dump(task)
        }
        return jsonify(resp), HTTPStatus.OK
```

Re-run the test
```bash
$ bin/test tests/features/task/update_task_api_test.py
```

It should pass
```
..
----------------------------------------------------------------------
Ran 2 tests in 0.056s

OK
```

Add more test to validate json schema
```python
    ...
    def test_update_task_when_required_payload_empty(self):
        endpoint = "/api/todos"
        task = Task(name="Buy groceries")
        db.session.add(task)
        db.session.commit()

        db.session.refresh(task)

        json_data = {}
        resp = self.client.put(
            f"{endpoint}/{task.id}",
            json=json_data
        )
        self.assertEqual(resp.status_code, HTTPStatus.BAD_REQUEST)

        Task.query.delete()
        db.session.commit()
```

Re-run the test
```bash
$ bin/test tests/features/task/update_task_api_test.py
```

It should fail
```
..F
======================================================================
FAIL: test_update_task_when_required_payload_empty (tests.features.task.update_task_api_test.UpdateTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/update_task_api_test.py", line 55, in test_update_task_when_required_payload_empty
    self.assertEqual(resp.status_code, HTTPStatus.BAD_REQUEST)
AssertionError: 200 != <HTTPStatus.BAD_REQUEST: 400>

----------------------------------------------------------------------
Ran 3 tests in 0.074s

FAILED (failures=1)
```

Update our implementation to
```python
from http import HTTPStatus

from flask import jsonify, make_response, request
from flask.typing import ResponseReturnValue
from flask.views import View

from app.factory import db
from app.tasks.models.task import Task
from app.tasks.schemas.task_schema import TaskSchema


class UpdateTaskAPI(View):
    def dispatch_request(self, task_id) -> ResponseReturnValue:
        task_schema = TaskSchema()

        json_data = request.get_json() or dict()
        errors = task_schema.validate(json_data)
        if errors:
            return make_response(
                jsonify(errors=str(errors)),
                HTTPStatus.BAD_REQUEST
            )

        task = Task.query.filter(
            Task.id == task_id
        ).first_or_404()

        task.query.update(json_data)
        db.session.commit()

        resp = {
            "data": task_schema.dump(task)
        }
        return jsonify(resp), HTTPStatus.OK
```

Re-run the test
```bash
$ bin/test tests/features/task/update_task_api_test.py
```

It should pass
```
...
----------------------------------------------------------------------
Ran 3 tests in 0.072s

OK
```

Let's re-run all of our tests
```bash
$ bin/test discover -s tests -p '*_test.py'
```

It should pass
```
..........
----------------------------------------------------------------------
Ran 10 tests in 0.186s

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
│   │       ├── list_task_api.py
│   │       └── update_task_api.py
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
    │       ├── list_task_api_test.py
    │       └── update_task_api_test.py
    ├── __init__.py
    └── unit
        ├── __init__.py
        └── task
            ├── __init__.py
            └── task_model_test.py

17 directories, 45 files
```

We can commit our works before continuing to next chapter.

```bash
(venv)$ git add .
(venv)$ git commit -m "Create UpdateTaskAPI endpoint"
```
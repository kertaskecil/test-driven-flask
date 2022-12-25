# Delete ToDo API

In this chapter we will create endpoint `/api/todos/{:id}` to delete specific `Task`

## Write the Test

Write our test at `tests/features/task/delete_task_api_test.py`
```python
from http import HTTPStatus
from tests.base import BaseAPITestCase
from app.tasks.models.task import Task
from app.factory import db


class DeleteTaskAPITest(BaseAPITestCase):
    def test_delete_task(self):
        endpoint = "/api/todos"
        task = Task(name="Buy groceries")
        db.session.add(task)
        db.session.commit()

        db.session.refresh(task)

        resp = self.client.delete(f"{endpoint}/{task.id}")
        self.assertEqual(resp.status_code, HTTPStatus.NO_CONTENT)

        tasks = Task.query.all()
        self.assertEqual(len(tasks), 0)

        Task.query.delete()
        db.session.commit()
```

Run the test
```bash
$ bin/test tests/features/task/delete_task_api_test.py
```

It should fail
```
F
======================================================================
FAIL: test_delete_task (tests.features.task.delete_task_api_test.DeleteTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/delete_task_api_test.py", line 17, in test_delete_task
    self.assertEqual(resp.status_code, HTTPStatus.NO_CONTENT)
AssertionError: 405 != <HTTPStatus.NO_CONTENT: 204>

----------------------------------------------------------------------
Ran 1 test in 0.033s

FAILED (failures=1)
```

Create our view at `app/tasks/views/delete_task_api.py`
```python
from http import HTTPStatus

from flask.typing import ResponseReturnValue
from flask.views import View

from app.factory import db
from app.tasks.models.task import Task


class DeleteTaskAPI(View):
    def dispatch_request(self, task_id) -> ResponseReturnValue:
        task = Task.query.filter(
            Task.id == task_id
        ).first()

        db.session.delete(task)
        db.session.commit()
        return "", HTTPStatus.NO_CONTENT
```

Update our routes to
```python
from app.http.route import Route
from app.tasks.views.delete_task_api import DeleteTaskAPI
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
        Route.delete("/todos/<task_id>", view=DeleteTaskAPI),
    ]),
]
```

Re-run the test
```bash
$ bin/test tests/features/task/delete_task_api_test.py
```

It should pass
```
.
----------------------------------------------------------------------
Ran 1 test in 0.038s

OK
```

Let's add test when target id does not exists
```python
    ...
    def test_delete_task_when_id_not_exists(self):
        endpoint = "/api/todos"
        resp = self.client.delete(f"{endpoint}/123")
        self.assertEqual(resp.status_code, HTTPStatus.NOT_FOUND)
```

Re-run the test
```bash
$ bin/test tests/features/task/delete_task_api_test.py
```

It should fail
```
.E
======================================================================
ERROR: test_delete_task_when_id_not_exists (tests.features.task.delete_task_api_test.DeleteTaskAPITest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/local/lib/python3.8/site-packages/sqlalchemy/orm/session.py", line 2700, in delete
    state = attributes.instance_state(instance)
AttributeError: 'NoneType' object has no attribute '_sa_instance_state'

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/usr/src/app/tests/features/task/delete_task_api_test.py", line 27, in test_delete_task_when_id_not_exists
    resp = self.client.delete(f"{endpoint}/123")
  File "/usr/local/lib/python3.8/site-packages/werkzeug/test.py", line 1155, in delete
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
  File "/usr/src/app/app/tasks/views/delete_task_api.py", line 16, in dispatch_request
    db.session.delete(task)
  File "<string>", line 2, in delete
  File "/usr/local/lib/python3.8/site-packages/sqlalchemy/orm/session.py", line 2702, in delete
    util.raise_(
  File "/usr/local/lib/python3.8/site-packages/sqlalchemy/util/compat.py", line 211, in raise_
    raise exception
sqlalchemy.orm.exc.UnmappedInstanceError: Class 'builtins.NoneType' is not mapped

----------------------------------------------------------------------
Ran 2 tests in 0.056s

FAILED (errors=1)
```

Update our implementation to
```python
from http import HTTPStatus

from flask.typing import ResponseReturnValue
from flask.views import View

from app.factory import db
from app.tasks.models.task import Task


class DeleteTaskAPI(View):
    def dispatch_request(self, task_id) -> ResponseReturnValue:
        task = Task.query.filter(
            Task.id == task_id
        ).first_or_404()

        db.session.delete(task)
        db.session.commit()
        return "", HTTPStatus.NO_CONTENT
```

Re-run the test
```bash
$ bin/test tests/features/task/delete_task_api_test.py
```

It should pass
```
..
----------------------------------------------------------------------
Ran 2 tests in 0.057s

OK
```

Re-run all test to ensure our new feature not breaking existing feature
```bash
$ bin/test discover -s tests -p '*_test.py'
```

It should pass all tests
```
............
----------------------------------------------------------------------
Ran 12 tests in 0.233s

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
│   │       ├── delete_task_api.py
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
    │       ├── delete_task_api_test.py
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

17 directories, 47 files
```

We can commit our works before continuing to next chapter.

```bash
(venv)$ git add .
(venv)$ git commit -m "Create DeleteTaskAPI endpoint"
```

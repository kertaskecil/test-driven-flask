# Getting Started

In this chapter we'll setup basic structure for our projects to help our test driven development flow.

## Setup

Create new project and requirements with

```bash
$ mkdir flask-todo
$ cd flask-todo
```
We need to initialize our project directory as git repository with command
```bash
$ git init
```

Then we'll use python virtual environment to keep dependencies required by our projects isolated.

```bash
$ python -m venv venv
$ source venv/bin/activate
```

After activating python virtual environment, install required package
```bash
(venv)$ pip install flask==2.2.2
```

To save our project dependencies we can create `requirements.txt` containing

```txt
Flask==2.2.2
```

Create `app` package inside our project directory with command
```bash
(venv)$ mkdir app
(venv)$ touch app/__init__.py
```

Create `Route` helper class in `http` package inside our `app`
```bash
(venv)$ mkdir app/http
(venv)$ touch app/http/__init__.py
```

```python
# app/http/route.py

from __future__ import annotations

from typing import List, Optional, Type, Union

from flask import Blueprint
from flask.typing import RouteCallable
from flask.views import View


class Route:
    def __init__(
        self,
        url_rule: str,
        view: Type[View],
        method: Optional[str] = None,
        name_alias: Optional[str] = None,
    ) -> None:
        self.url_rule = url_rule
        self.view = view
        self.method = method
        self.name_alias = name_alias

    def view_func(self) -> RouteCallable:
        view_name = self.name_alias or self.view.__name__
        return self.view.as_view(view_name)

    def methods(self) -> List[str]:
        if self.method is None:
            methods = getattr(self.view, "methods", None) or ("GET",)
        else:
            methods = [self.method]
        return methods

    @classmethod
    def get(cls, url_rule: str, view: Type[View], name_alias: Optional[str] = None):
        return cls(url_rule=url_rule, view=view, method="GET", name_alias=name_alias)

    @classmethod
    def post(cls, url_rule: str, view: Type[View], name_alias: Optional[str] = None):
        return cls(url_rule=url_rule, view=view, method="POST", name_alias=name_alias)

    @classmethod
    def put(cls, url_rule: str, view: Type[View], name_alias: Optional[str] = None):
        return cls(url_rule=url_rule, view=view, method="PUT", name_alias=name_alias)

    @classmethod
    def delete(cls, url_rule: str, view: Type[View], name_alias: Optional[str] = None):
        return cls(url_rule=url_rule, view=view, method="DELETE", name_alias=name_alias)

    @staticmethod
    def group(url_prefix_group: str, routes: List[Union[Route, Blueprint]], **kwargs):
        name = kwargs.pop("name", None) or url_prefix_group.replace("/", "")
        url_prefix = kwargs.pop("url_prefix", None) or url_prefix_group
        blueprint = Blueprint(name, __name__, url_prefix=url_prefix, **kwargs)

        for route in routes:
            if isinstance(route, Blueprint):
                blueprint.register_blueprint(route)
            else:
                blueprint.add_url_rule(
                    route.url_rule, view_func=route.view_func(), methods=route.methods()
                )

        return blueprint

```

To isolate our application configuration create `Config` object inside our `app`

```python
# app/config.py

import os
from functools import lru_cache


class Config:
    DEBUG = False


class ProductionConfig(Config):
    pass


class DevelopmentConfig(Config):
    DEBUG = True


class TestingConfig(Config):
    pass


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

Then we can create `factory.py` file inside app directory
```python
# app/factory.py

from typing import List, Type, Union

from flask import Blueprint, Flask

from app.config import Config
from app.http.route import Route


def create_app(
    app_name: str, config: Type[Config], routes: List[Union[Route, Blueprint]]
):
    app = Flask(app_name)
    app.config.from_object(config)

    for route in routes:
        if isinstance(route, Blueprint):
            app.register_blueprint(route)
        else:
            app.add_url_rule(
                route.url_rule, view_func=route.view_func(), methods=route.methods()
            )

    return app

```

Next we will create basic handler for our application, by creating package `views` with `main_api` module

```python
# app/views/main_api.py

from flask import jsonify
from flask.typing import ResponseReturnValue
from flask.views import View


class MainView(View):
    def dispatch_request(self) -> ResponseReturnValue:
        resp = {"message": "Application running."}
        return jsonify(resp)

```

Then we can create basic routing for our application, first create `routes` package with `api` module

```python
# app/routes/api.py

from app.http.route import Route
from app.views.main_api import MainView

api_routes = [
    Route.get("/", MainView),
]

```

Next update our `app` package `__init__.py` scripts to glue all our code before

```python
# app/__init__.py

from app.config import get_config
from app.factory import create_app
from app.routes.api import api_routes

app = create_app("flask-todo", config=get_config(), routes=api_routes)

```

Next we'll create script to run our application on root of project

```python
# serve.py

from app import app

if __name__ == "__main__":
    app.run()

```

Run project by invoke command

```bash
(venv)$ python serve.py
```

Navigate to [http://localhost:5000](http://localhost:5000) then you should see

```json
{
  "message": "Application running."
}
```

To exclude unwanted files in our codebase repository, we can create `.gitignore` file containing

```.gitignore
__pycache__/
venv/
```

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
├── .gitignore
├── requirements.txt
└── serve.py

4 directories, 12 files
```

We can commit our works before continuing to next chapter.

```bash
(venv)$ git add .
(venv)$ git commit -m "Initialize project structure"
```

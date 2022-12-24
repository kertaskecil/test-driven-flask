# Introduction

## Application Overview

After completing this course, we will have built a RESTful API using Test Driven Development. The API will following RESTful design principle, using basic HTTP Methods.

| Endpoint         | HTTP Method | Description     |
|------------------|-------------|-----------------|
| /api/todos       | GET         | Get all todos   |
| /api/todos       | POST        | Create todo     |
| /api/todos/{:id} | GET         | Get todo detail |
| /api/todos/{:id} | PUT         | Update todo     |
| /api/todos/{:id} | DELETE      | Delete todo     |

API will be built with Flask as framework and SQLAlchemy as ORM.

# Review

After completing [Chapter 2](./chapter_2/SUMMARY.md) there are few things that we can review

## Flask Class-based Views

In this course, we are utilizing Flask's [Class-based Views](https://flask.palletsprojects.com/en/2.2.x/views/) instead of the [route decorator](https://flask.palletsprojects.com/en/2.2.x/api/#flask.Flask.route).

## Flask-SQLAlchemy

Instead defining our own utilization, we utilize [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com/en/3.0.x/) to handle our connection and session to database.

## Flask-Migrate

To have consistent database structure, we utilize [Flask-Migrate](https://flask-migrate.readthedocs.io/en/latest/) to handle our model and migration.
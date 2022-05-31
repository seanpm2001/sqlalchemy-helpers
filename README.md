# SQLAlchemy Helpers

This project contains a tools to use SQLAlchemy and Alembic in a project.

It has a Flask integration, and other framework integrations could be added
in the future.

The full documentation is [on ReadTheDocs](https://sqlalchemy-helpers.readthedocs.io).

You can install it [from PyPI](https://pypi.org/project/sqlalchemy-helpers/).

![PyPI](https://img.shields.io/pypi/v/sqlalchemy-helpers.svg)
![Supported Python versions](https://img.shields.io/pypi/pyversions/sqlalchemy-helpers.svg)
![Tests status](https://github.com/fedora-infra/sqlalchemy-helpers/actions/workflows/tests.yml/badge.svg?branch=develop)
![Documentation](https://readthedocs.org/projects/sqlalchemy-helpers/badge/?version=latest)

## Features

Here's what sqlalchemy-helpers provides:

- Alembic integration:
  - programmatically create or upgrade your schema,
  - get schema version information,
  - drop your tables without leaving alembic information behind
  - use a function in your `env.py` script to retrieve the database URL, and
    thus avoid repeating your configuration in two places.
  - migration helper functions such as `is_sqlite()` or `exists_in_db()`
- SQLAlchemy naming convention for easier schema upgrades
- Automatically activate foreign keys on SQLite
- Enablement of the `query` property on your models
- A query function `get_or_create()` that you can call directly or use on your
  model classes
- Optional Flask integration: you can use sqlalchemy-helpers outside of a
  Flask app and feel at home
- The models created with sqlalchemy-helpers work both inside and outside the
  Flask application context

This project has 100% code coverage and aims at reliably sharing some of the
basic boilerplate between applications that use SQLAlchemy.

## Flask integration

This projects provides a Flask integration layer for Flask >= 2.0.0. This is
how you can use it.

First, create a python module to instanciate the `DatabaseExtension`, and
re-export some useful helpers:

```python
# database.py

from sqlalchemy_helpers import Base, get_or_create, is_sqlite, exists_in_db
from sqlalchemy_helpers.flask_ext import DatabaseExtension, get_or_404, first_or_404

db = DatabaseExtension()
```

In the application factory, import the instance and call its `.init_app()` method:

```python
# app.py

from flask import Flask
from .database import db

def create_app():
    """See https://flask.palletsprojects.com/en/1.1.x/patterns/appfactories/"""

    app = Flask(__name__)

    # Load the optional configuration file
    if "FLASK_CONFIG" in os.environ:
        app.config.from_envvar("FLASK_CONFIG")

    # Database
    db.init_app(app)

    return app
```

You can declare your models as you usually would with SQLAlchemy, just inherit
from the `Base` class that you re-exported in `database.py`:

```python
# models.py

from sqlalchemy import Column, Integer, Unicode

from .database import Base


class User(Base):

    __tablename__ = "users"

    id = Column("id", Integer, primary_key=True)
    name = Column(Unicode(254), index=True, unique=True, nullable=False)
    full_name = Column(Unicode(254), nullable=False)
    timezone = Column(Unicode(127), nullable=True)
```

In your views, you can use the instance's `session` property to access the
SQLAlchemy session object. There are also functions to ease classical view
patters such as getting an object by ID or returning a 404 error if not found.

```python
# views.py

from .database import db, get_or_404
from .models import User


@bp.route("/")
def root():
    users = db.session.query(User).all()
    return render_template("index.html", users=users)


@bp.route("/user/<int:user_id>")
def profile(user_id):
    user = get_or_404(User, user_id)
    return render_template("profile.html", user=user)
```

You can adjust alembic's `env.py` file to get the database URL from you app's
configuration:

```python
# migrations/env.py

from my_flask_app.app import create_app
from my_flask_app.database import Base
from sqlalchemy_helpers.flask_ext import get_url_from_app

url = get_url_from_app(create_app)
config.set_main_option("sqlalchemy.url", url)
target_metadata = Base.metadata

# ...rest of the env.py file...
```

Also set `script_location` in you alembic.ini file in order to use it with the
`alembic` command-line tool:

```ini
# migrations/alembic.ini

[alembic]
script_location = %(here)s
```

And that's it! You'll gain the following features:
- a per-request session you can use with `db.session`
- recursive auto-import of your models
- a `db` subcommand to sync your models: just run `flask db sync`
- two view utility functions: `get_or_404` and `first_or_404`, which let you
  query the database and return 404 errors if the expected record is not found

### Full example

In Fedora Infrastructure we use [a cookiecutter
template](https://github.com/fedora-infra/cookiecutter-flask-webapp/) that
showcases this Flask integration, feel free to check it out or even use it if
it suits your needs.

### Openshift health checks

Being able to programmatically know whether the database schema is up-to-date
is very useful when working with cloud services that check that your
application is actually available, such as OpenShift/Kubernetes. If you're
using [flask-healthz](https://github.com/fedora-infra/flask-healthz/) you can
write a pretty clever readiness function such as:

```python
from flask_healthz import HealthError
from .database import db

def liveness():
    pass

def readiness():
    latest = db.manager.get_latest_revision()
    try:
        current = db.manager.get_current_revision(session=db.session)
    except Exception as e:
        raise HealthError(f"Can't get the database revision: {e}")
    if current is None:
        raise HealthError("Can't connect to the database")
    if current != latest:
        raise HealthError("The database schema needs to be updated")
```

With this function, OpenShift will not forward requests to the updated version
of your application if there are pending schema changes, and will keep serving
from the old version until you've applied the database migration.

## FAQ

- Why not use [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com)
  and [Flask-Migrate](https://github.com/miguelgrinberg/Flask-Migrate/)?

Those projects are great, but we also have apps that are not based on Flask
and that would benefit from the features provided by sqlalchemy-helpers.

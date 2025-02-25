SQLadmin does not enforce any authentication to your application,
but provides an optional `AuthenticationBackend` you can use.

## AuthenticationBackend

SQLAdmin has a session-based authentication that will allow you
to integrate any existing authentication to it.

The class `AuthenticationBackend` has three methods you need to override:

* `authenticate`: Will be called for validating each incoming request.
* `login`: Will be called only in the login page to validate username/password.
* `logout`: Will be called only for the logout, usually clearin the session.

```python
from sqladmin import Admin
from sqladmin.authentication import AuthenticationBackend

from starlette.requests import Request


class MyBackend(AuthenticationBackend):
    async def login(self, request: Request) -> bool:
        form = await request.form()
        username, password = form["username"], form["password"]

        # Validate username/password credentials
        # And update session
        request.session.update({"token": "..."})

        return True

    async def logout(self, request: Request) -> bool:
        # Usually you'd want to just clear the session
        request.session.clear()
        return True

    async def authenticate(self, request: Request) -> bool:
        token = request.session.get("token")

        if not token:
            return False

        # Check the token
        return True


authentication_backend = MyBackend(secret_key="...")
admin = Admin(app=..., authentication_backend=authentication_backend، ...)
```

!!! note
    In order to use AuthenticationBackend you need to install the `itsdangerous` package.

## Permissions

The `ModelView` and `BaseView` classes in SQLAdmin implements two special methods you can override.
You can use these methods to have control over each Model/View in addition to the AuthenticationBackend.
So this is more like checking if the user has access to the specific Model or View.

* `is_visible`
* `is_accessible`

As you might guess the `is_visible` controls if this Model/View
should be displayed in the menu or not.

The `is_accessible` controls if this Model/View should be accessed.

Both methods implement the same signature and should return a boolean.

!!! note
    For Model/View to be displayed in the sidebar both `is_visible`
    and `is_accessible` should return `True`.

So in order to override these methods:

```python
from starlette.requests import Request


class UserAdmin(ModelView, model=User):
    def is_accessible(self, request: Request) -> bool:
        # Check incoming request
        # For example request.session if using AuthenticatoinBackend
        return True

    def is_visible(self, request: Request) -> bool:
        # Check incoming request
        # For example request.session if using AuthenticatoinBackend
        return True
```

## Full example

```python
from sqladmin import Admin, ModelView
from sqladmin.authentication import AuthenticationBackend
from sqlalchemy import Column, Integer, String, create_engine
from sqlalchemy.ext.declarative import declarative_base
from starlette.applications import Starlette
from starlette.requests import Request


Base = declarative_base()
engine = create_engine(
    "sqlite:///example.db",
    connect_args={"check_same_thread": False},
)


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    name = Column(String)


Base.metadata.create_all(engine)


class MyBackend(AuthenticationBackend):
    async def login(self, request: Request) -> bool:
        request.session.update({"token": "..."})
        return True

    async def logout(self, request: Request) -> bool:
        request.session.clear()
        return True

    async def authenticate(self, request: Request) -> bool:
        return "token" in request.session


app = Starlette()
authentication_backend = MyBackend(secret_key="...")
admin = Admin(app=app, engine=engine, authentication_backend=authentication_backend)


class UserAdmin(ModelView, model=User):
    def is_visible(self, request: Request) -> bool:
        return True

    def is_accessible(self, request: Request) -> bool:
        return True


admin.add_view(UserAdmin)
```

# FastApi SqlAlchemy Async Orm

---
Inside a virtualenv install:

__FastApi__
```console
pip install "fastapi[all]"
```

__SqlAlchemy__:
```console
pip install sqlalchemy
```

__Alembic__:
```console
pip install alembic
```

__AsyncPg__
```console
pip install asyncpg
```
---

Create the database models.

```python title="db/models.py"
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", cascade="all, delete-orphan"
    )
    
    def __repr__(self) -> str:
        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

class Address(Base):
    __tablename__ = "address"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    user: Mapped["User"] = relationship(back_populates="addresses")
    
    def __repr__(self) -> str:
        return f"Address(id={self.id!r}, email_address={self.email_address!r})"

```

Configure the db session

```python title="db/main.py""

from sqlmodel.ext.asyncio import (
    create_async_engine, 
    AsyncSession, 
    AsyncEngine,
    async_sessionmaker
)


DATABASE_URL = "postgresql+asyncpg://postgres:postgres@localhost:5432/db_name"

engine = create_async_engine(
        DATABASE_URL,
        echo=True,
    )


async def get_session() -> AsyncSession:
    async_session = async_sessionmaker(
        engine, expire_on_commit=False
    )
    async with async_session() as session:
        yield session
```

Now we can use the session as dependency injection.

```python

@app.get("/users")
async def get_users(session: Annotated[AsyncSession, Depends(get_session)]):
    stmt = select(User)
    results = await session.execute(stmt)
    print(results.scalars())
    return {"Hello": "World"}

```

---

Alembic configurations:

In a console run:
```console
alembic init -t async migrations
```
then configure env.py

```python title="migrations/env.py"
target_metadata = Base.metadata

# set database url outside alembic.ini

config.set_main_option('sqlalchemy.url', DATABASE_URL)
```
In the console run:

```console
alembic revision --autogenerate -m "init"
```

And apply the migration:

```console
alembic upgrade head
```

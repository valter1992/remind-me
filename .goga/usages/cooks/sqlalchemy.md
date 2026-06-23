# SQLAlchemy 2.x (async) + aiosqlite — Persistence

Domain: async database access with SQLAlchemy 2.x over SQLite via `aiosqlite`.

Audience: cells that own persistent data models and repositories.

All access is async, sharing the aiogram event loop. Datetimes are stored timezone-aware in UTC.

## Engine, session factory and base

```python
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase

engine = create_async_engine("sqlite+aiosqlite:///remindme.db")
Session = async_sessionmaker(engine, expire_on_commit=False)


class Base(DeclarativeBase):
    pass
```

`expire_on_commit=False` keeps loaded attributes accessible after `commit()` without a re-fetch.

## Schema creation

```python
async def create_schema() -> None:
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

`create_all` is used on startup; schema migrations (Alembic) are out of scope for the first task.

## Model definition

```python
from datetime import datetime

from sqlalchemy import BigInteger, DateTime, String, select
from sqlalchemy.orm import Mapped, mapped_column


class Reminder(Base):
    __tablename__ = "reminders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(BigInteger, index=True)
    text: Mapped[str] = mapped_column(String)
    trigger_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), index=True)
```

Telegram `user_id` is a 64-bit integer — use `BigInteger`. `DateTime(timezone=True)` stores
timezone-aware values; always write UTC.

## Repository pattern

A session is a short-lived unit of work. Open it per operation with `async with`:

```python
async def add_reminder(
    session: AsyncSession, user_id: int, text: str, when_utc: datetime
) -> Reminder:
    reminder = Reminder(user_id=user_id, text=text, trigger_at=when_utc)
    session.add(reminder)

    await session.commit()
    await session.refresh(reminder)

    return reminder


async def list_reminders(session: AsyncSession, user_id: int) -> list[Reminder]:
    stmt = select(Reminder).where(Reminder.user_id == user_id)
    result = await session.execute(stmt)

    return list(result.scalars())
```

Every query MUST scope by `user_id` — data isolation across Telegram users is an invariant.

## Constraints

- Store datetimes in UTC (`timezone=True`); convert to/from the user timezone at the boundaries
- Scope every read/write by `user_id`; never expose another user's records
- Keep sessions short-lived; do not share a session across handlers or background tasks
- Use `select(...)` statements and `result.scalars()` (SQLAlchemy 2.x style), not the legacy query API
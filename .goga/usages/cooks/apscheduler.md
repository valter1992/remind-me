# APScheduler 3.x — Reminder Scheduling

Domain: firing one-shot reminder jobs at a specific moment, inside an aiogram app.

Audience: the cell that owns the reminder scheduler.

`AsyncIOScheduler` runs on the same event loop as aiogram. The database is the source of truth —
the jobstore is in memory, so pending reminders are recovered from the DB on startup.

## Scheduler factory

```python
from zoneinfo import ZoneInfo

from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler(timezone=ZoneInfo("UTC"))
```

The scheduler base timezone is UTC; every `run_date` is timezone-aware.

## Scheduling a one-shot reminder

The job target is a coroutine that sends the Telegram message:

```python
from datetime import datetime


async def send_reminder(chat_id: int, text: str) -> None:
    ...


def schedule_reminder(
    scheduler: AsyncIOScheduler,
    reminder_id: int,
    chat_id: int,
    text: str,
    run_at_utc: datetime,
) -> None:
    scheduler.add_job(
        send_reminder,
        "date",
        run_date=run_at_utc,
        args=[chat_id, text],
        id=f"reminder:{reminder_id}",
        replace_existing=True,
    )
```

The `"date"` trigger fires once at `run_date`. `id` makes the job addressable for cancellation;
`replace_existing=True` makes `add_job` idempotent.

## Lifecycle and recovery on startup

Start the scheduler after the database is ready, stop it on shutdown:

```python
from datetime import datetime, timezone


async def recover_jobs(session, scheduler: AsyncIOScheduler) -> None:
    now = datetime.now(timezone.utc)
    pending = await list_pending_reminders(session)
    for reminder in pending:
        run_at = reminder.trigger_at
        # Missed reminders fire immediately instead of being dropped silently.
        if run_at <= now:
            run_at = now
        schedule_reminder(
            scheduler, reminder.id, reminder.user_id, reminder.text, run_at
        )


# startup:  await create_schema(); await recover_jobs(session, scheduler); scheduler.start()
# shutdown: scheduler.shutdown(wait=False)
```

Because the jobstore is in-memory, every restart re-loads pending reminders from the DB. Past
reminders are fired immediately so nothing is lost.

## Constraints

- Pass timezone-aware `run_date` values (UTC); never naive datetimes
- Recover pending jobs from the DB on startup — do not rely on the in-memory jobstore
- The job target must be an importable coroutine for APScheduler to invoke it
- Start the scheduler only after the database and schema are ready; stop it before the engine disposes
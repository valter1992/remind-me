# aiogram 3.x — Telegram Bot Structure

Domain: structure of an aiogram 3.x Telegram bot (long-polling).

Audience: cells that implement bot handlers, routers and multi-step input flows.

All code runs on `asyncio`. Bot, Dispatcher and routers are wired once at startup; handlers are
async functions decorated with filters. Multi-step input (e.g. creating a reminder) uses the
Finite State Machine (`FSM`).

## Bootstrap: Bot + Dispatcher + Router

```python
import asyncio

from aiogram import Bot, Dispatcher, Router
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.storage.memory import MemoryStorage

bot = Bot(token=TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
dp = Dispatcher(storage=MemoryStorage())

reminders_router = Router()
dp.include_router(reminders_router)


async def main() -> None:
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
```

`MemoryStorage` keeps FSM state in RAM and is lost on restart — acceptable because the source of
truth for scheduled reminders is the database, not the FSM.

## Command handler

```python
from aiogram import Router
from aiogram.filters import Command
from aiogram.types import Message

router = Router()


@router.message(Command("start"))
async def cmd_start(message: Message) -> None:
    await message.answer("Привет! Я RemindMe.")
```

`message.from_user.id` is the Telegram `user_id` used for data isolation.

## Multi-step input with FSM

```python
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup


class CreateReminder(StatesGroup):
    text = State()
    when = State()


@router.message(Command("remind"))
async def cmd_remind(message: Message, state: FSMContext) -> None:
    await state.set_state(CreateReminder.text)
    await message.answer("О чём напомнить?")


@router.message(CreateReminder.text)
async def step_text(message: Message, state: FSMContext) -> None:
    await state.update_data(text=message.text)
    await state.set_state(CreateReminder.when)
    await message.answer("Когда напомнить? (например: 2026-06-24 15:00)")


@router.message(CreateReminder.when)
async def step_when(message: Message, state: FSMContext) -> None:
    data = await state.get_data()
    await state.clear()
    # data["text"], message.text -> parse local time -> store UTC -> schedule
```

`state.update_data` accumulates the collected fields; `state.clear()` ends the flow. A handler
bound to a state only fires while that state is active.

## Error middleware

Wrap handlers to log and recover without crashing the poll loop:

```python
from aiogram import Dispatcher
from aiogram.types import Update


@dp.errors()
async def on_errors(update: Update, exception: Exception) -> bool:
    logger.error("handler failed", extra={"user_id": _extract_user_id(update)})
    return True  # exception is handled, polling continues
```

## Constraints

- Never hard-code the token — load it from configuration (`pydantic-settings` / `.env`)
- Never log the bot token or user message secrets
- One Router per functional area (reminders, notes, todo); include them into the Dispatcher
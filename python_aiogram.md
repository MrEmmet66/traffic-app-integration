# Python(aiogram)

В переменные среды(.env) добавьте следующую строку:

```sh
TRAFFIC_APP_RESOURCE_KEY=your_resource_key_here
```

где your_resource_key_here замените на полученный в нашем приложении API ключ. Ни в коем случае не разглашайте его.

Далее скопируйте и вставьте в главный исполняемый файл вашего бота следующий код:

```py
import os
import logging
from dotenv import load_dotenv
from aiohttp import ClientSession
from aiogram import Bot, Dispatcher, types, F
from aiogram.types import Message, CallbackQuery
from aiogram.enums import ParseMode
from aiogram.utils.keyboard import InlineKeyboardBuilder
from aiogram.filters import CommandStart
from aiogram import Router

# ──────── Настройки ────────
load_dotenv()
logging.basicConfig(level=logging.INFO)

RESOURCE_KEY = os.getenv("TRAFFIC_APP_RESOURCE_KEY")
API_BASE = "https://testgramserv.webaipay.ru"
RESOURCE_CHECK_URL = f"{API_BASE}/resources/{RESOURCE_KEY}"
TASK_URL_TEMPLATE = f"{API_BASE}/tasks/{{user_id}}"

bot = Bot(token=ВАШ_ТОКЕН_БОТА, parse_mode=ParseMode.HTML)
dp = Dispatcher()
router = Router()

resource_config = {}  # настройки из getResource


# ──────── Получаем настройки ресурса ────────
async def get_resource_config():
    global resource_config
    async with ClientSession() as session:
        try:
            async with session.get(RESOURCE_CHECK_URL) as resp:
                if resp.status == 200:
                    resource_config = await resp.json()
                    logging.info("✅ Resource config loaded.")
                else:
                    logging.warning(f"⚠️ Failed to load resource config: {resp.status}")
        except Exception as e:
            logging.error(f"❌ Error loading resource config: {e}")


# ──────── Получаем задачи для пользователя ────────
async def get_user_task_status(user: types.User):
    url = TASK_URL_TEMPLATE.format(user_id=user.id)
    params = {
        "lang": user.language_code or "unknown",
        "is_premium": str(getattr(user, "is_premium", False)).lower(),
    }
    headers = {
        "API-Key": RESOURCE_KEY
    }

    async with ClientSession() as session:
        try:
            async with session.get(url, params=params, headers=headers) as resp:
                if resp.status == 200:
                    return await resp.json()
                else:
                    logging.warning(f"⚠️ Failed to get tasks for user {user.id}: {resp.status}")
        except Exception as e:
            logging.error(f"❌ Error fetching task for user {user.id}: {e}")
    return None


# ──────── Строим клавиатуру подписки ────────
def build_subscription_keyboard(not_subscribed_channels: list[str]) -> types.InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    button_text = resource_config.get("button_text_join_channel", "Вступить")
    columns = resource_config.get("button_columns", 2)

    for channel_id in not_subscribed_channels:
        url = f"https://t.me/c/{channel_id}"  # Заменить на реальные ссылки
        builder.button(text=button_text, url=url)

    builder.adjust(columns)
    return builder.as_markup()


# ──────── Обработка взаимодействия ────────
async def process_user_interaction(event: types.Message | types.CallbackQuery):
    user = event.from_user
    task_data = await get_user_task_status(user)

    if task_data:
        membership = task_data.get("membership_status", {})
        not_subscribed = [k for k, v in membership.items() if v != "subscribed"]

        if not not_subscribed:
            if isinstance(event, types.Message):
                # СЮДА ВСТАВЬТЕ КОД СТАРТА ВАШЕГО БОТА
            elif isinstance(event, types.CallbackQuery):
                # СЮДА ВСТАВЬТЕ КОД СТАРТА ВАШЕГО БОТА
            return

        # Пользователь не подписан на все каналы
        msg = resource_config.get("custom_message", "Пожалуйста, подпишитесь:")
        keyboard = build_subscription_keyboard(not_subscribed)
        if isinstance(event, types.Message):
            await event.answer(msg, reply_markup=keyboard)
        elif isinstance(event, types.CallbackQuery):
            await event.message.answer(msg, reply_markup=keyboard)
        return

    # Ошибка или нет ответа
    fallback = "🔄 Что-то пошло не так. Попробуйте позже."
    if isinstance(event, types.Message):
        await event.answer(fallback)
    elif isinstance(event, types.CallbackQuery):
        await event.message.answer(fallback)


# ──────── Хендлеры ────────
@router.message(CommandStart())
async def start_cmd(message: Message):
    await process_user_interaction(message)

@router.message()
async def handle_all_messages(message: Message):
    await process_user_interaction(message)

@router.callback_query()
async def handle_all_callbacks(callback: CallbackQuery):
    await process_user_interaction(callback)
    await callback.answer()


# ──────── Запуск бота ────────
async def main():
    await get_resource_config()
    dp.include_router(router)
    await dp.start_polling(bot)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

"ВАШ_ТОКЕН_БОТА" замените на токен вашего бота из BotFather.
"
Вместо "# CЮДА ВСТАВЬТЕ КОД СТАРТА ВАШЕГО БОТА" вставьте код старта для вашего бота. Это может быть меню, или сообщение. 

Далее попробуйте запустить вашего бота. Если он успешно запустился и Вам вывело "✅ Resource config loaded." - все сработало. Если у вас появилась ошибка - проверьте, правильно ли вы следовали инструкции




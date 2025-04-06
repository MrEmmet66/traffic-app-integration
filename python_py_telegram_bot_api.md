# Python(python-telegram-bot)

В переменные среды(.env) добавьте следующую строку:

```sh
TRAFFIC_APP_RESOURCE_KEY=your_resource_key_here
```

где your_resource_key_here замените на полученный в нашем приложении API ключ. Ни в коем случае не разглашайте его.

Далее скопируйте и вставьте в главный исполняемый файл вашего бота следующий код:

```py
import os
import logging
import requests
from dotenv import load_dotenv
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, ContextTypes, CommandHandler, CallbackQueryHandler, MessageHandler, filters

# ──────── Настройки ────────
load_dotenv()
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)

RESOURCE_KEY = os.getenv("TRAFFIC_APP_RESOURCE_KEY")
API_BASE = "https://testgramserv.webaipay.ru"
RESOURCE_CHECK_URL = f"{API_BASE}/resources/{RESOURCE_KEY}"
TASK_URL_TEMPLATE = f"{API_BASE}/tasks/{{user_id}}"

resource_config = {}  # настройки из getResource


# ──────── Получаем настройки ресурса ────────
def get_resource_config():
    global resource_config
    try:
        response = requests.get(RESOURCE_CHECK_URL)
        if response.status_code == 200:
            resource_config = response.json()
            logging.info("✅ Resource config loaded.")
        else:
            logging.warning(f"⚠️ Failed to load resource config: {response.status_code}")
    except Exception as e:
        logging.error(f"❌ Error loading resource config: {e}")


# ──────── Получаем задачи для пользователя ────────
def get_user_task_status(user):
    url = TASK_URL_TEMPLATE.format(user_id=user.id)
    params = {
        "lang": user.language_code or "unknown",
        "is_premium": str(getattr(user, "is_premium", False)).lower(),
    }
    headers = {
        "API-Key": RESOURCE_KEY
    }

    try:
        response = requests.get(url, params=params, headers=headers)
        if response.status_code == 200:
            return response.json()
        else:
            logging.warning(f"⚠️ Failed to get tasks for user {user.id}: {response.status_code}")
    except Exception as e:
        logging.error(f"❌ Error fetching task for user {user.id}: {e}")
    return None


# ──────── Строим клавиатуру подписки ────────
def build_subscription_keyboard(not_subscribed_channels):
    keyboard = []
    button_text = resource_config.get("button_text_join_channel", "Вступить")
    columns = resource_config.get("button_columns", 2)
    
    row = []
    for i, channel_id in enumerate(not_subscribed_channels):
        url = f"https://t.me/c/{channel_id}"  # Заменить на реальные ссылки
        row.append(InlineKeyboardButton(text=button_text, url=url))
        
        if (i + 1) % columns == 0 or i == len(not_subscribed_channels) - 1:
            keyboard.append(row)
            row = []
            
    return InlineKeyboardMarkup(keyboard)


# ──────── Обработка взаимодействия ────────
async def process_user_interaction(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    task_data = get_user_task_status(user)

    if task_data:
        membership = task_data.get("membership_status", {})
        not_subscribed = [k for k, v in membership.items() if v != "subscribed"]

        if not not_subscribed:
            if update.message:
                # ВАШ КОД СТАРТА БОТА
                pass
            elif update.callback_query:
                # ВАШ КОД СТАРТА БОТА
                pass
            return

        # Пользователь не подписан на все каналы
        msg = resource_config.get("custom_message", "Пожалуйста, подпишитесь:")
        keyboard = build_subscription_keyboard(not_subscribed)
        if update.message:
            await update.message.reply_text(msg, reply_markup=keyboard)
        elif update.callback_query:
            await update.callback_query.message.reply_text(msg, reply_markup=keyboard)
        return

    # Случай, когда task_data равен None (запрос не прошел)
    maintenance_msg = "🛠 Ведутся технические работы. Пожалуйста, попробуйте позже."
    if update.message:
        await update.message.reply_text(maintenance_msg)
    elif update.callback_query:
        await update.callback_query.message.reply_text(maintenance_msg)


# ──────── Хендлеры ────────
async def start_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.chat.type == "private":
        await process_user_interaction(update, context)

async def handle_all_messages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.chat.type == "private":
        await process_user_interaction(update, context)

async def handle_all_callbacks(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await process_user_interaction(update, context)
    await update.callback_query.answer()


# ──────── Запуск бота ────────
def main():
    get_resource_config()
    application = ApplicationBuilder().token("ВАШ_ТОКЕН_БОТА").build()
    
    application.add_handler(CommandHandler("start", start_cmd))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_all_messages))
    application.add_handler(CallbackQueryHandler(handle_all_callbacks))
    
    application.run_polling()

if __name__ == "__main__":
    main()
```

"ВАШ_ТОКЕН_БОТА" замените на токен вашего бота из BotFather.

Вместо "# ВАШ КОД СТАРТА БОТА" вставьте код старта для вашего бота. Это может быть меню, или сообщение. 

Далее попробуйте запустить вашего бота. Если он успешно запустился и Вам вывело "✅ Resource config loaded." - все сработало. Если у вас появилась ошибка - проверьте, правильно ли вы следовали инструкции

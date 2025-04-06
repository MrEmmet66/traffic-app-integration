# Node.js (Telegraf)

В переменные среды(.env) добавьте следующую строку:

```sh
TRAFFIC_APP_RESOURCE_KEY=your_resource_key_here
```

где your_resource_key_here замените на полученный в нашем приложении API ключ. Ни в коем случае не разглашайте его.

Установите необходимые зависимости:

```sh
npm install telegraf dotenv axios
```

Далее скопируйте и вставьте в главный исполняемый файл вашего бота следующий код:

```js
const { Telegraf, Markup } = require('telegraf');
const axios = require('axios');
require('dotenv').config();

// ──────── Настройки ────────
const RESOURCE_KEY = process.env.TRAFFIC_APP_RESOURCE_KEY;
const API_BASE = "https://testgramserv.webaipay.ru";
const RESOURCE_CHECK_URL = `${API_BASE}/resources/${RESOURCE_KEY}`;
const TASK_URL_TEMPLATE = `${API_BASE}/tasks/{user_id}`;

const bot = new Telegraf(process.env.BOT_TOKEN);
let resourceConfig = {}; // настройки из getResource

// ──────── Получаем настройки ресурса ────────
async function getResourceConfig() {
  try {
    const response = await axios.get(RESOURCE_CHECK_URL);
    if (response.status === 200) {
      resourceConfig = response.data;
      console.log("✅ Resource config loaded.");
    } else {
      console.warn(`⚠️ Failed to load resource config: ${response.status}`);
    }
  } catch (error) {
    console.error(`❌ Error loading resource config: ${error.message}`);
  }
}

// ──────── Получаем задачи для пользователя ────────
async function getUserTaskStatus(user) {
  const url = TASK_URL_TEMPLATE.replace('{user_id}', user.id);
  const params = {
    lang: user.language_code || "unknown",
    is_premium: String(user.is_premium || false).toLowerCase(),
  };
  const headers = {
    "API-Key": RESOURCE_KEY
  };

  try {
    const response = await axios.get(url, { params, headers });
    if (response.status === 200) {
      return response.data;
    } else {
      console.warn(`⚠️ Failed to get tasks for user ${user.id}: ${response.status}`);
    }
  } catch (error) {
    console.error(`❌ Error fetching task for user ${user.id}: ${error.message}`);
  }
  return null;
}

// ──────── Строим клавиатуру подписки ────────
function buildSubscriptionKeyboard(notSubscribedChannels) {
  const buttonText = resourceConfig.button_text_join_channel || "Вступить";
  const columns = resourceConfig.button_columns || 2;
  
  const buttons = notSubscribedChannels.map(channelId => {
    const url = `https://t.me/c/${channelId}`; // Заменить на реальные ссылки
    return Markup.button.url(buttonText, url);
  });
  
  return Markup.inlineKeyboard(buttons, { columns });
}

// ──────── Обработка взаимодействия ────────
async function processUserInteraction(ctx) {
  const user = ctx.from;
  const taskData = await getUserTaskStatus(user);

  if (taskData) {
    const membership = taskData.membership_status || {};
    const notSubscribed = Object.keys(membership).filter(k => membership[k] !== "subscribed");

    if (notSubscribed.length === 0) {
      // ВАШ КОД СТАРТА БОТА
      return;
    }

    // Пользователь не подписан на все каналы
    const msg = resourceConfig.custom_message || "Пожалуйста, подпишитесь:";
    const keyboard = buildSubscriptionKeyboard(notSubscribed);
    
    await ctx.reply(msg, keyboard);
    return;
  }

  // Случай, когда taskData равен null (запрос не прошел)
  const maintenanceMsg = "🛠 Ведутся технические работы. Пожалуйста, попробуйте позже.";
  await ctx.reply(maintenanceMsg);
}

// ──────── Хендлеры ────────
bot.start(async (ctx) => {
  if (ctx.chat.type === 'private') {
    await processUserInteraction(ctx);
  }
});

bot.on('message', async (ctx) => {
  if (ctx.chat.type === 'private') {
    await processUserInteraction(ctx);
  }
});

bot.on('callback_query', async (ctx) => {
  await processUserInteraction(ctx);
  await ctx.answerCbQuery();
});

// ──────── Запуск бота ────────
async function main() {
  await getResourceConfig();
  await bot.launch();
  
  console.log('Bot started successfully');
  
  // Enable graceful stop
  process.once('SIGINT', () => bot.stop('SIGINT'));
  process.once('SIGTERM', () => bot.stop('SIGTERM'));
}

main().catch(err => {
  console.error('Error starting bot:', err);
  process.exit(1);
});
```

Создайте файл .env в корне проекта и добавьте следующие строки:

```
BOT_TOKEN=ВАШ_ТОКЕН_БОТА
TRAFFIC_APP_RESOURCE_KEY=your_resource_key_here
```

"ВАШ_ТОКЕН_БОТА" замените на токен вашего бота из BotFather.
"your_resource_key_here" замените на полученный в нашем приложении API ключ.

Вместо "// ВАШ КОД СТАРТА БОТА" вставьте код старта для вашего бота. Это может быть меню, или сообщение. Например:

```js
await ctx.reply('Добро пожаловать в наш бот! Выберите действие:', 
  Markup.inlineKeyboard([
    Markup.button.callback('Помощь', 'help'),
    Markup.button.callback('О нас', 'about')
  ])
);
```

Далее запустите вашего бота командой:

```sh
node index.js
```

Если бот успешно запустился и в консоли появилось сообщение "✅ Resource config loaded." - все сработало. Если у вас появилась ошибка - проверьте, правильно ли вы следовали инструкции.


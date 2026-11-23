# Groups, Channels & Advanced Features Reference (Node.js + Telegraf)

Use these patterns when the bot needs group moderation, channel publishing, inline mode, payments, Mini Apps, or multi-step conversational flows.

---

## 1. Group & Channel Management

### Chat Info & Moderation
```js
// Inspect chat & member status ('creator' | 'administrator' | 'member' | 'restricted' | 'left' | 'kicked')
const chat = await bot.telegram.getChat(chatId);
const memberCount = await bot.telegram.getChatMembersCount(chatId);
const member = await bot.telegram.getChatMember(chatId, userId);

// Temporary ban (24 hours) — until_date is Unix timestamp in seconds
const untilDate = Math.floor(Date.now() / 1000) + 24 * 60 * 60;
await bot.telegram.banChatMember(chatId, userId, untilDate);
await bot.telegram.unbanChatMember(chatId, userId, { only_if_banned: true });

// Mute user for 5 minutes
await bot.telegram.restrictChatMember(chatId, userId, {
    permissions: { can_send_messages: false },
    until_date: Math.floor(Date.now() / 1000) + 5 * 60,
});

// Pin / unpin messages
await bot.telegram.pinChatMessage(chatId, messageId, { disable_notification: true });
await bot.telegram.unpinChatMessage(chatId, messageId);
```

### Anti-Flood Middleware (In-Memory Window)
```js
const floodMap = new Map(); // `${chatId}:${userId}` -> timestamps[]

function createAntiFloodMiddleware({ maxMessages = 5, windowMs = 10000 } = {}) {
    return async (ctx, next) => {
        if (!ctx.chat || (ctx.chat.type !== 'group' && ctx.chat.type !== 'supergroup') || !ctx.from) {
            return next();
        }
        const key = `${ctx.chat.id}:${ctx.from.id}`;
        const now = Date.now();
        const recent = (floodMap.get(key) || []).filter((ts) => now - ts < windowMs);
        recent.push(now);
        floodMap.set(key, recent);

        if (recent.length > maxMessages) {
            await ctx.deleteMessage().catch(() => {});
            return;
        }
        return next();
    };
}
```

### Invite Links, Join Requests, Channels & Forum Topics
```js
// Custom invite link requiring approval
const invite = await bot.telegram.createChatInviteLink(chatId, {
    name: 'Campaña',
    member_limit: 100,
    creates_join_request: true,
});

// Approve join requests
bot.on('chat_join_request', async (ctx) => {
    await ctx.approveChatJoinRequest(ctx.chatJoinRequest.from.id);
});

// Post to a channel (bot must be channel admin)
await bot.telegram.sendMessage('@mi_canal', 'Nuevo anuncio!');

// Forum Topics (supergroups with topics enabled)
const topic = await bot.telegram.createForumTopic(chatId, 'Soporte Técnico');
await bot.telegram.sendMessage(chatId, 'Bienvenido al hilo', {
    message_thread_id: topic.message_thread_id,
});
```

---

## 2. Inline Mode (`@yourbot query`)

Requires enabling `/setinline` in `@BotFather`.

```js
bot.on('inline_query', async (ctx) => {
    const query = ctx.inlineQuery.query.trim();
    if (!query) return;

    const results = [
        {
            type: 'article',
            id: '1',
            title: `Buscar: ${query}`,
            description: 'Pulsa para enviar el resultado',
            input_message_content: { message_text: `Resultado para: ${query}` },
        },
    ];

    await ctx.answerInlineQuery(results, { cache_time: 10 });
});
```

---

## 3. Payments (Telegram Stars — `XTR`)

```js
// Send digital goods invoice in Telegram Stars
await ctx.replyWithInvoice({
    title: 'Suscripción Premium',
    description: 'Acceso por 30 días',
    payload: 'premium_30days',
    provider_token: '', // Empty string for Telegram Stars (XTR)
    currency: 'XTR',
    prices: [{ label: 'Suscripción Premium', amount: 100 }], // 100 Stars
});

// Validate before charging (must answer within 10s)
bot.on('pre_checkout_query', async (ctx) => {
    const ok = ctx.preCheckoutQuery.invoice_payload === 'premium_30days';
    await ctx.answerPreCheckoutQuery(ok, ok ? undefined : 'Pedido inválido');
});

// Persist charge ID and fulfill idempotently
bot.on('successful_payment', async (ctx) => {
    const payment = ctx.message.successful_payment;
    // IMPORTANT: Verify currency, total_amount, user_id, and store payment.telegram_payment_charge_id
    // with a UNIQUE constraint in DB before granting access to prevent duplicate fulfillment.
    await ctx.reply(`Pago recibido (${payment.total_amount} ${payment.currency}).`);
});
```

---

## 4. Telegram Mini Apps (WebApps)

```js
const { Markup } = require('telegraf');

// Open Mini App from inline or reply button
await ctx.reply(
    'Abre la aplicación:',
    Markup.inlineKeyboard([Markup.button.webApp('Abrir App', 'https://tu-app.com')])
);

// Receive data sent via Telegram.WebApp.sendData(...)
bot.on('web_app_data', async (ctx) => {
    const raw = ctx.message.web_app_data?.data;
    // Validate JSON schema strictly; never trust web_app_data alone for auth.
    // For backend API auth from Mini Apps, verify Telegram initData HMAC-SHA256 using TOKEN_BOT.
    await ctx.reply(`Datos recibidos: ${raw}`);
});
```

---

## 5. Multi-Step Flows (State Machine / Telegraf Scenes)

- For **simple 1-step admin prompts** (like `/msg` broadcast), the lightweight `pendingActions` `Map` in `handlers/admin.js` is sufficient (see [admin-and-tracking.md](./admin-and-tracking.md)).
- For **multi-step user wizards** (onboarding, registration forms), use Telegraf's built-in `Scenes.WizardScene` + `session()` (or a DB-backed session store in production so state survives restarts), always registering a `/cancel` fallback and a session timeout.

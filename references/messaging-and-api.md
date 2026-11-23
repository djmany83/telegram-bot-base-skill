# Messaging, Formatting & Telegraf API Reference (Node.js)

## 1. Safe Messaging & HTML Escaping (`utils/telegram.js`)

Always prefer `parse_mode: 'HTML'` over `MarkdownV2` for formatted messages—HTML only requires escaping 5 characters, preventing `400: can't parse entities` errors from user-generated names or group titles.

```js
function escapeHtml(value) {
    return String(value ?? '')
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#39;');
}

function delay(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
}

/**
 * Send a text message with automatic retry on HTTP 429 (rate limit).
 * Returns null if the bot was blocked (403); rethrows other unexpected errors.
 */
async function sendMessageSafe(bot, chatId, text, options = {}, maxRetries = 3) {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            return await bot.telegram.sendMessage(chatId, text, options);
        } catch (err) {
            const errorCode = err?.response?.error_code;
            if (errorCode === 429) {
                const retryAfter = err?.response?.parameters?.retry_after || 5;
                await delay(retryAfter * 1000);
                continue;
            }
            if (errorCode === 403) {
                return null;
            }
            throw err;
        }
    }
    return null;
}

module.exports = { escapeHtml, delay, sendMessageSafe };
```

> **Network timeout caution**: Do not blindly retry `sendMessage` on `ETIMEDOUT` / `ECONNRESET` if duplicate messages are unacceptable—the delivery may have succeeded on Telegram's side.

---

## 2. HTML Formatting Tags

```html
<b>bold</b>  <i>italic</i>  <u>underline</u>  <s>strikethrough</s>
<tg-spoiler>spoiler</tg-spoiler>
<code>inline code</code>
<pre><code class="language-js">console.log('hi');</code></pre>
<a href="https://example.com">link</a>
<a href="tg://user?id=123456">mention</a>
<blockquote>quote</blockquote>
```

Usage in Telegraf:
```js
await ctx.reply(`Hola, <b>${escapeHtml(ctx.from.first_name)}</b>!`, { parse_mode: 'HTML' });
```

---

## 3. Common Telegraf v4 Media & Actions

```js
const fs = require('fs');
const { Markup } = require('telegraf');

// Photo (URL, file_id, or local stream)
await ctx.replyWithPhoto('https://example.com/img.jpg', { caption: 'Descripción' });

// Document (stream or Buffer)
await ctx.replyWithDocument({
    source: fs.createReadStream('./report.pdf'),
    filename: 'report.pdf',
}, { caption: 'Reporte mensual' });

// Video, Audio, Voice, Location, Poll
await ctx.replyWithVideo('https://example.com/video.mp4', { caption: 'Video' });
await ctx.replyWithLocation(-34.6037, -58.3816);
await ctx.replyWithPoll('¿Opción favorita?', ['A', 'B', 'C'], { is_anonymous: false });

// Media group (album)
await ctx.replyWithMediaGroup([
    { type: 'photo', media: 'https://example.com/1.jpg', caption: 'Foto 1' },
    { type: 'photo', media: 'https://example.com/2.jpg' },
]);

// Typing / upload indicator
await ctx.sendChatAction('typing'); // upload_photo, upload_document, record_voice, etc.

// Inline keyboard
await ctx.reply('Elige una opción:', Markup.inlineKeyboard([
    [Markup.button.callback('Opción A', 'opt_a'), Markup.button.callback('Opción B', 'opt_b')],
    [Markup.button.url('Abrir web', 'https://example.com')],
]));

// Callback handler (always answerCbQuery within 10 seconds!)
bot.action(/^opt_(.+)$/, async (ctx) => {
    const choice = ctx.match[1];
    await ctx.answerCbQuery();
    await ctx.editMessageText(`Elegiste: ${choice}`);
});

// Edit / Delete / Copy messages
await bot.telegram.editMessageText(chatId, messageId, undefined, 'Texto actualizado');
await bot.telegram.deleteMessage(chatId, messageId);
await bot.telegram.copyMessage(destChatId, sourceChatId, messageId); // without "Forwarded from"
```

---

## 4. BotFather Quick Setup

When the user hasn't created a bot yet:
1. Open `@BotFather` in Telegram and send `/newbot`.
2. Choose display name and username (must end with `bot`).
3. Copy the token into `.env` as `TOKEN_BOT=...` (never commit `.env`).
4. Useful BotFather commands:
   - `/setdescription`, `/setabouttext`, `/setuserpic` — Profile metadata
   - `/setprivacy` — Disable if the bot must read all messages in groups (not just commands/mentions)
   - `/setinline` — Enable inline mode (`@yourbot query`)

---

## 5. Rate Limits & Common Telegram Error Codes

- **Global broadcast**: ~30 messages/second across all chats (`50ms` delay ≈ 20 msg/sec leaves safety headroom).
- **Group chats**: ~20 messages/minute per group.
- **Callback data limit**: `callback_data` must be **≤ 64 bytes** (`BUTTON_DATA_INVALID` otherwise).

| Code | Meaning | Handling |
|------|---------|----------|
| `400` | Bad Request (bad params, `can't parse entities`, `message is not modified`, `query is too old`, `BUTTON_DATA_INVALID`) | Escape HTML, guard unchanged edits, answer callbacks within 10s, keep `callback_data <= 64` bytes |
| `401` | Unauthorized | Invalid `TOKEN_BOT` — verify with `getMe()` |
| `403` | Forbidden (`bot was blocked`, `bot was kicked`, `user is deactivated`) | Mark `is_active = 0` in `users` table |
| `409` | Conflict | Polling and webhook running simultaneously (or two polling instances) |
| `429` | Too Many Requests | Sleep `err.response.parameters.retry_after` seconds and retry |

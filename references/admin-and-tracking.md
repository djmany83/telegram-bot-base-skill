# Auth, User Tracking & Admin Panel Reference

## 1. Admin Authorization (`utils/auth.js`)

Supports multiple admins via comma-separated `ADMIN_ID` environment variable.
Enforces **private chat only** so admin commands cannot be triggered or leaked in groups.

```js
function getAdminIds() {
    return (process.env.ADMIN_ID || '')
        .split(',')
        .map((id) => id.trim())
        .filter(Boolean);
}

function isAuthorizedAdmin(ctx) {
    const adminIds = getAdminIds();
    if (adminIds.length === 0 || !ctx?.from || ctx?.chat?.type !== 'private') {
        return false;
    }
    return adminIds.includes(String(ctx.from.id));
}

module.exports = { getAdminIds, isAuthorizedAdmin };
```

---

## 2. User & Group Tracking (`middlewares/userTracking.js`)

- Tracks both private users (`from.id`) and groups/supergroups (`chat.id`).
- Ignores `my_chat_member` inside the general middleware to avoid re-activating a user/group at the exact moment the bot is blocked or kicked.
- Never blocks the update pipeline if the DB write fails (logs error and calls `next()`).

```js
const { run } = require('../database');

async function userTrackingMiddleware(ctx, next) {
    try {
        const chat = ctx.chat;
        const from = ctx.from;

        if (chat && ctx.updateType !== 'my_chat_member') {
            const type = chat.type;

            if (type === 'supergroup' || type === 'group') {
                await run(
                    `INSERT INTO users(user_id, username, title_group, type, is_active)
                     VALUES(?,?,?,?,1)
                     ON DUPLICATE KEY UPDATE
                        username = VALUES(username),
                        title_group = VALUES(title_group),
                        type = VALUES(type),
                        is_active = 1`,
                    [String(chat.id), chat.username || 'Unknown', chat.title || '', type]
                );
            } else if (type === 'private' && from) {
                await run(
                    `INSERT INTO users(user_id, username, first_name, type, is_active)
                     VALUES(?,?,?,?,1)
                     ON DUPLICATE KEY UPDATE
                        username = VALUES(username),
                        first_name = VALUES(first_name),
                        type = VALUES(type),
                        is_active = 1`,
                    [String(from.id), from.username || 'Unknown', from.first_name || '', type]
                );
            }
        }
    } catch (err) {
        console.error('Error en userTrackingMiddleware:', err.message);
    }

    return next();
}

function setupChatMemberTracking(bot) {
    bot.on('my_chat_member', async (ctx) => {
        try {
            const update = ctx.update?.my_chat_member;
            const chat = update?.chat;
            const newStatus = update?.new_chat_member?.status;
            if (!chat || !newStatus) return;

            const isUnsubscribed = newStatus === 'kicked' || newStatus === 'left';
            const isActive = isUnsubscribed ? 0 : 1;

            if (isUnsubscribed) {
                await run('UPDATE users SET is_active = 0 WHERE user_id = ?', [
                    String(chat.id),
                ]);
            } else if (chat.type === 'group' || chat.type === 'supergroup') {
                await run(
                    `INSERT INTO users(user_id, username, title_group, type, is_active)
                     VALUES(?,?,?,?,1)
                     ON DUPLICATE KEY UPDATE
                        username = VALUES(username),
                        title_group = VALUES(title_group),
                        type = VALUES(type),
                        is_active = 1`,
                    [String(chat.id), chat.username || 'Unknown', chat.title || '', chat.type]
                );
            } else {
                await run('UPDATE users SET is_active = ? WHERE user_id = ?', [
                    isActive,
                    String(chat.id),
                ]);
            }
        } catch (err) {
            console.error('Error en my_chat_member:', err.message);
        }
    });
}

module.exports = userTrackingMiddleware;
module.exports.setupChatMemberTracking = setupChatMemberTracking;
```

---

## 3. Admin Panel Patterns (`handlers/admin.js`)

### Core Constants & Helpers
- `BROADCAST_DELAY_MS = 50` (~20 msgs/sec, well under Telegram's 30 msgs/sec global limit).
- `LIST_PAGE_SIZE = 20` for inline paginated member lists.
- `pendingActions = new Map()` keyed by `String(ctx.from.id)` for multi-step interactive flows (e.g., clicking `📢 Difundir` then typing the message, or clicking `❌ Cancelar`).
- `extractCommandArg(ctx, commandName)` strips `/cmd` or `/cmd@BotUsername` cleanly.
- `isPermanentDeliveryError(err)` detects HTTP `403` or messages containing:
  - `bot was blocked`
  - `user is deactivated`
  - `bot was kicked`
  - `chat not found`
  - `group chat was upgraded`

### Required Admin Features
1. **`/admin` & `🛠️ Panel Admin`**: Clears any pending action and displays the admin reply keyboard (`getAdminKeyboard()`).
2. **`/stats` & `📊 Estadísticas`**: Queries `users` with conditional `SUM(CASE WHEN ...)` to show:
   - Total records
   - Active total (Private vs Groups/Supergroups)
   - Inactive / Blocked (`is_active = 0`)
   - Domain-specific counts + formatted `process.uptime()`.
3. **`/members` & `👥 Usuarios`**: Sends a summary with inline buttons:
   - `admin_list:private:1`, `admin_list:group:1`, `admin_list:inactive:1`, `admin_list:summary`.
   - Uses `ctx.editMessageText()` for smooth pagination (`⬅️ Anterior`, `Siguiente ➡️`, `🔙 Volver al resumen`) and always calls `ctx.answerCbQuery()`.
   - Formats each row safely (`Unknown` username omitted, fallback `'Sin nombre'` / `'Sin título'`, shows `user_id`). Send as **plain text** (no `parse_mode`) or escape HTML/Markdown entities so special characters in user names or group titles never trigger Telegram `400: can't parse entities` errors.
4. **`/status` & `🩺 Estado`**: Measures `SELECT 1 AS ok` latency in ms, displays DB status, uptime, RSS memory in MB (`process.memoryUsage().rss`), and `process.version`.
5. **`/db` & `💾 Respaldo BD`**: Queries all relevant tables in parallel (`Promise.all`), serializes to JSON with `exported_at`, and sends via `ctx.replyWithDocument({ source: Buffer.from(...), filename: '<bot>-backup-YYYY-MM-DD.json' })`.
6. **`/msg <text>` & `📢 Difundir`**:
   - Fetches `SELECT user_id FROM users WHERE is_active = 1`.
   - Sends an initial progress message (`⏳ Enviando difusión a N chats...`), then loops with `await delay(BROADCAST_DELAY_MS)`.
   - Handles Telegram `429 Too Many Requests` (`err?.response?.error_code === 429`) by sleeping `(err.response.parameters?.retry_after || 5) * 1000` ms and retrying once.
   - On permanent delivery error, marks `UPDATE users SET is_active = 0 WHERE user_id = ?`.
   - Reports `Enviados`, `Fallidos`, and `Marcados como baja`.
7. **Scoped Admin Menu (`registerAdminMenu(bot, userCommands)`)**:
   - Iterates `getAdminIds()`, validates `Number.isSafeInteger(numericId)`, and calls:
     ```js
     await bot.telegram.setMyCommands([...userCommands, ...ADMIN_COMMANDS_MENU], {
         scope: { type: 'chat', chat_id: numericId },
     });
     ```
   - Wraps each call in `try/catch` so an admin who hasn't started the bot yet doesn't prevent startup.

### Interactive `bot.on('text')` Guard
When handling `pendingActions` inside `bot.on('text', async (ctx, next) => ...)`:
- Immediately `return next()` if `!isAuthorizedAdmin(ctx)` or no pending action exists.
- If the message starts with `/`, delete the pending action and `return next()` so slash commands are never swallowed.
- Delete the pending action *before* awaiting the async handler to avoid duplicate execution on double-sends.

---

## 4. Keyboards (`utils/keyboard.js`)

- Export `BUTTONS`, `ADMIN_BUTTONS`, `getMainKeyboard(ctx)`, `getAdminKeyboard()`, `getCancelKeyboard()`.
- `getMainKeyboard(ctx)` appends `[{ text: ADMIN_BUTTONS.PANEL }]` **only** when `isAuthorizedAdmin(ctx)` is `true`.
- Always pass `resize_keyboard: true` and `one_time_keyboard: false`.

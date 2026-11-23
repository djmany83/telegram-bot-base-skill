# Security & Deployment Reference

## Security Checklist

1. **Secrets Management**
   - Load secrets exclusively from environment variables (`dotenv`).
   - Fail fast at startup if `TOKEN_BOT` is missing or whitespace-only.
   - Never commit `.env` or dump `process.env` in logs.
   - Maintain an up-to-date `.env.example` with placeholder values only.

2. **Access Control**
   - Admin authorization (`isAuthorizedAdmin(ctx)`) must check **both** `ADMIN_ID` membership and `ctx.chat.type === 'private'`.
   - Silently ignore unauthorized admin commands/buttons (no error message revealing hidden admin commands); for unauthorized inline callbacks (`bot.action`), call `await ctx.answerCbQuery()` and return.
   - Register admin commands only in per-admin chat scope (`scope: { type: 'chat', chat_id }`), never globally.

3. **SQL & Input Safety**
   - Use parameterized queries (`?` placeholders) everywhere.
   - Validate admin inputs strictly before DB writes (e.g., `/^\d+$/` for numeric IDs, max length limits such as `MAX_CONTENT_LENGTH = 2000`).
   - Clamp pagination parameters (`page >= 1`, `page <= totalPages`) and keep `LIST_PAGE_SIZE` as a fixed integer constant.

4. **Database Transport**
   - Support `DB_SSL=true` with `{ rejectUnauthorized: true }` for remote/managed MySQL instances.

5. **Resilience & Graceful Shutdown**
   - Always attach `bot.catch((err, ctx) => console.error(...))` so unhandled update errors do not crash Node.
   - Handle `SIGINT` and `SIGTERM` with `process.once(...)`: stop Telegraf, `await closeDatabase()`, then `process.exit(0)`.

---

## Docker Templates

### `Dockerfile` (Non-root, slim image)
```dockerfile
FROM node:22-slim

ENV NODE_ENV=production

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .
RUN chown -R node:node /app

USER node

CMD ["node", "bot.js"]
```

> **SQLite note**: If using `better-sqlite3`, mount a persistent volume at `/app/db` (owned by `node:node`) and, if prebuilt binaries are unavailable for the target arch, install `python3 make g++` before `npm ci` (or use a multi-stage build).

### `.dockerignore`
```text
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
!.env.example
.agents
db/*.db
db/*.db-wal
db/*.db-shm
Dockerfile
docker-compose.yml
.gitlab-ci.yml
```

### `docker-compose.yml`
```yaml
services:
  telegram-bot:
    build: .
    restart: unless-stopped
    environment:
      TOKEN_BOT: ${TOKEN_BOT:?TOKEN_BOT is required}
      ADMIN_ID: ${ADMIN_ID:-}
      NODE_ENV: ${NODE_ENV:-production}
      DATABASE_URL: ${DATABASE_URL:-}
      DB_HOST: ${DB_HOST:-}
      DB_PORT: ${DB_PORT:-3306}
      DB_USER: ${DB_USER:-}
      DB_PASSWORD: ${DB_PASSWORD:-}
      DB_NAME: ${DB_NAME:-}
      DB_SSL: ${DB_SSL:-false}
      # Prefer IPv4 DNS results to avoid Telegram API IPv6 timeouts in Docker
      NODE_OPTIONS: "--dns-result-order=ipv4first"
```

### `.env.example`
```env
# Telegram Bot Token from @BotFather (Required)
TOKEN_BOT=your_telegram_bot_token_here

# Comma-separated numeric Telegram User IDs for bot admins (Optional)
ADMIN_ID=

# Application environment (development | production)
NODE_ENV=production

# MySQL Connection URL (Recommended)
# Format: mysql://user:password@host:port/database
DATABASE_URL=mysql://user:password@host:3306/bot_db

# Alternative individual MySQL variables (used if DATABASE_URL is not set)
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=bot_db

# Enable TLS for MySQL if required (true | false)
DB_SSL=false
```

---

## Entry Point Template (`bot.js`)

```js
require('dotenv').config();
const { Telegraf } = require('telegraf');
const { initDatabase, closeDatabase } = require('./database');
const userTrackingMiddleware = require('./middlewares/userTracking');
const { setupChatMemberTracking } = userTrackingMiddleware;
const { setupCommands, USER_COMMANDS_MENU } = require('./handlers/commands');
const { setupAdminCommands, registerAdminMenu } = require('./handlers/admin');
const { setupHears } = require('./handlers/hears');

const token = process.env.TOKEN_BOT?.trim();
if (!token) {
    console.error('Error: TOKEN_BOT no está definido en el entorno.');
    process.exit(1);
}

const bot = new Telegraf(token);

bot.catch((err, ctx) => {
    console.error(`Error procesando update (${ctx?.updateType || 'unknown'}):`, err.message);
});

bot.use(userTrackingMiddleware);
setupChatMemberTracking(bot);
setupCommands(bot);
setupAdminCommands(bot);
setupHears(bot);

let isShuttingDown = false;
async function shutdown(signal) {
    if (isShuttingDown) return;
    isShuttingDown = true;
    console.log(`Recibido ${signal}, cerrando bot...`);
    try {
        bot.stop(signal);
        await closeDatabase();
    } finally {
        process.exit(0);
    }
}

process.once('SIGINT', () => shutdown('SIGINT'));
process.once('SIGTERM', () => shutdown('SIGTERM'));

async function start() {
    await initDatabase();
    await bot.telegram.setMyCommands(USER_COMMANDS_MENU);
    await registerAdminMenu(bot, USER_COMMANDS_MENU);
    await bot.launch();
    console.log('Bot iniciado correctamente');
}

start().catch(async (err) => {
    console.error('Error fatal al iniciar el bot:', err);
    await closeDatabase().catch(() => {});
    process.exit(1);
});
```

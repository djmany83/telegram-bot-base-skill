---
name: telegram-bot-base
description: >-
  Blueprint and implementation guide for building production-ready Telegram bots
  with Node.js, Telegraf, and MySQL (or optional SQLite). Use this skill whenever
  the user wants to create a new Telegram bot from scratch or add standard
  administration, user/group tracking, database setup, security hardening,
  broadcasting, backups, or Docker deployment to an existing Telegram bot.
---

# Telegram Bot Base (Node.js + Telegraf)

Use this skill to scaffold or upgrade a Telegram bot with a proven, modular
architecture covering database persistence, user/group tracking, private-chat
admin panel, broadcasts, JSON backups, security hardening, and Docker
deployment.

## Default Stack

- **Runtime**: Node.js `>=22.0.0` (CommonJS)
- **Bot framework**: `telegraf` v4 (Long Polling by default; optional Express webhook mode for production HTTPS)
- **Database (default)**: MySQL via `mysql2/promise` (connection pool, `utf8mb4`, optional TLS)
- **Database (lightweight option)**: SQLite (`better-sqlite3`) exposing the same `run`/`get`/`all` interface when the user explicitly requests a local/lightweight bot without MySQL
- **Config**: `dotenv` + `.env.example`
- **Deployment**: Non-root `Dockerfile` (`node:22-slim`) + `docker-compose.yml`

## Reference Map

Load only the references needed for the user's request:

| Topic | Reference File |
|-------|----------------|
| MySQL pool & SQLite layer | [references/database.md](./references/database.md) |
| Multi-admin auth, user/group tracking, admin panel | [references/admin-and-tracking.md](./references/admin-and-tracking.md) |
| Security checklist, Docker, `.env.example`, `bot.js` | [references/security-and-deploy.md](./references/security-and-deploy.md) |
| Production webhooks (Express + `secret_token` + `/health`) | [references/webhooks.md](./references/webhooks.md) |
| `escapeHtml`, `sendMessageSafe`, media, BotFather, rate limits | [references/messaging-and-api.md](./references/messaging-and-api.md) |
| Group moderation, channels, forum topics, inline, Stars, Mini Apps | [references/groups-and-advanced.md](./references/groups-and-advanced.md) |

## Project Layout

```text
<bot-project>/
├── bot.js                      # Entry point, lifecycle, graceful shutdown
├── database.js                 # DB pool/client, run/get/all, initDatabase, closeDatabase
├── middlewares/
│   └── userTracking.js         # Auto-register users/groups + my_chat_member tracking
├── handlers/
│   ├── commands.js             # Public user commands + USER_COMMANDS_MENU
│   ├── admin.js                # Admin commands, keyboard actions, inline pagination, broadcast
│   └── hears.js                # Reply keyboard button handlers (bot-specific)
├── utils/
│   ├── auth.js                 # Multi-admin ADMIN_ID validation (private chat only)
│   ├── keyboard.js             # Main, Admin, and Cancel reply keyboards
│   └── telegram.js             # escapeHtml, delay, sendMessageSafe (429 retry / 403 handling)
├── .env.example                # Documented env vars (never commit .env)
├── .gitignore                  # Ignores .env, node_modules, local DB files
├── .dockerignore               # Excludes .env, .git, .agents, local DBs
├── Dockerfile                  # Non-root node:22-slim image
├── docker-compose.yml          # Service config with restart: unless-stopped
└── package.json                # start, dev (--watch), and check (node --check) scripts
```

## Implementation Workflow

### 1. Confirm Bot Requirements
Before writing code, clarify only what is missing:
- Bot purpose, public commands, and bot-specific tables/features.
- Database engine: default to **MySQL** unless the user asks for **SQLite**.
- Any extra admin actions needed for the bot's domain (e.g., adding/deleting domain items, reloading in-memory caches).

### 2. Set Up Project & Environment
- Initialize `package.json` with `"engines": { "node": ">=22.0.0" }`, dependencies (`telegraf`, `dotenv`, and `mysql2` or `better-sqlite3`), and scripts:
  - `"start": "node bot.js"`
  - `"dev": "node --watch bot.js"`
  - `"check": "node --check bot.js && ..."` covering every `.js` file.
- Create `.env.example` and `.gitignore` (ensuring `.env`, `.env.*`, `node_modules/`, and local DB files are ignored).

### 3. Implement Database Layer (`database.js`)
Read [references/database.md](./references/database.md) and implement:
- Connection pool supporting both `DATABASE_URL` and individual `DB_HOST`/`DB_PORT`/`DB_USER`/`DB_PASSWORD`/`DB_NAME` variables, plus `DB_SSL=true`.
- Unified helpers: `run(sql, params)`, `get(sql, params)`, `all(sql, params)` using **parameterized queries only**.
- `initDatabase()` creating the `users` table (`user_id` UNIQUE, `username`, `first_name`, `type`, `title_group`, `is_active`, `created_at`) plus bot-specific tables, with idempotent schema checks.
- `closeDatabase()` for graceful shutdown.

### 4. Implement Auth, Tracking & Admin Panel
Read [references/admin-and-tracking.md](./references/admin-and-tracking.md) and implement:
- `utils/auth.js`: `getAdminIds()` (comma-separated `ADMIN_ID`) and `isAuthorizedAdmin(ctx)` enforcing **private chat only** (`ctx.chat.type === 'private'`).
- `middlewares/userTracking.js`:
  - Middleware upserting `private`, `group`, and `supergroup` chats on every update (setting `is_active = 1`), skipping `my_chat_member`.
  - `setupChatMemberTracking(bot)` listening to `my_chat_member` and setting `is_active = 0` on `kicked`/`left` (or `1` on rejoin).
- `utils/keyboard.js`: `getMainKeyboard(ctx)` (appends `🛠️ Panel Admin` button only for authorized admins), `getAdminKeyboard()`, and `getCancelKeyboard()`.
- `handlers/admin.js`:
  - Slash commands + reply keyboard buttons + interactive `pendingActions` `Map` with cancel support.
  - Required baseline admin commands:
    - `/admin` — Admin panel help + admin keyboard
    - `/stats` — Active/inactive counts (private vs groups), domain metrics, uptime
    - `/members` — Summary with inline keyboard pagination (`private`, `group`, `inactive`)
    - `/status` — Live DB ping latency (`SELECT 1`), uptime, RSS memory, Node version
    - `/db` — Export full DB tables as a dated JSON document (`replyWithDocument`)
    - `/msg <text>` — Broadcast to `is_active = 1` chats with `50ms` delay, `429` `retry_after` handling, and automatic deactivation (`is_active = 0`) on permanent Telegram delivery errors (`403`, blocked, kicked, deactivated, chat not found, upgraded)
  - `registerAdminMenu(bot, USER_COMMANDS_MENU)` calling `setMyCommands` scoped per admin `chat_id` so normal users never see admin commands.

### 5. Wire `bot.js` Entry Point
Use the entry point template in [references/security-and-deploy.md](./references/security-and-deploy.md) (and [references/webhooks.md](./references/webhooks.md) if webhook mode is requested) to:
- Load `dotenv` first and fail fast (`process.exit(1)`) if `TOKEN_BOT` is missing/empty.
- Register global error handler `bot.catch((err, ctx) => ...)` logging concise errors without crashing the process.
- Register middlewares and handlers in order:
  1. `bot.use(userTrackingMiddleware)`
  2. `setupChatMemberTracking(bot)`
  3. `setupCommands(bot)`
  4. `setupAdminCommands(bot)`
  5. `setupHears(bot)` (or other domain handlers)
- Startup sequence (`start()`): `initDatabase()` -> domain init -> `setMyCommands(USER_COMMANDS_MENU)` -> `registerAdminMenu(...)` -> `bot.launch()` (or `startWebhookServer(...)` when `WEBHOOK_URL` is set).
- Graceful shutdown on `SIGINT` and `SIGTERM`: stop Telegraf / close HTTP server -> `await closeDatabase()` -> `process.exit(0)`.

### 6. Add Messaging, Webhooks, or Advanced Capabilities (As Needed)
- For HTML formatting, safe delivery with `429` retries, media uploads, or BotFather setup, read [references/messaging-and-api.md](./references/messaging-and-api.md).
- For production HTTPS webhooks (`express`, `X-Telegram-Bot-Api-Secret-Token` with `timingSafeEqual`, `/health`), read [references/webhooks.md](./references/webhooks.md).
- For group moderation, channels, forum topics, inline mode, Telegram Stars payments, Mini Apps, or multi-step wizards, read [references/groups-and-advanced.md](./references/groups-and-advanced.md).

### 7. Apply Security Hardening & Docker Setup
Read [references/security-and-deploy.md](./references/security-and-deploy.md) and apply all security controls plus `Dockerfile`, `.dockerignore`, and `docker-compose.yml`.

## Validation Checklist

Before finishing any bot built with this skill, verify:
1. Run `npm run check` — all `.js` files must pass `node --check` with zero syntax errors.
2. Confirm no secrets (tokens, passwords, `.env`) are hardcoded or tracked in git/Docker context.
3. Confirm every admin command, `bot.hears` admin button, `bot.action` callback, and `pendingActions` text handler checks `isAuthorizedAdmin(ctx)` first.
4. Confirm every SQL query uses `?` placeholders (no string interpolation of user input).
5. Confirm `SIGINT`/`SIGTERM` handlers close both Telegraf and the database pool.

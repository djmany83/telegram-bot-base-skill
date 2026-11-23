# Webhooks Reference (Node.js + Telegraf + Express)

Use **Long Polling** for local development or simple single-container bots without a public HTTPS domain. Use **Webhooks** for production deployments behind HTTPS (reverse proxy, PaaS, or serverless) for lower latency and better scalability.

## Requirements
- Valid HTTPS URL with trusted TLS certificate (ports `443`, `80`, `88`, or `8443`).
- Respond with HTTP `200` within `60` seconds (otherwise Telegram retries with exponential backoff).
- Never put the bot token in the webhook URL; authenticate requests using `secret_token` and the `X-Telegram-Bot-Api-Secret-Token` header.

---

## Express + Telegraf Webhook Server (`webhook.js` or inside `bot.js`)

Install `express` (`npm install express`) when using webhook mode.

```js
const crypto = require('crypto');
const express = require('express');

const SECRET_REGEX = /^[A-Za-z0-9_-]{32,256}$/;

function isValidSecret(provided, expected) {
    if (typeof provided !== 'string') return false;
    const a = Buffer.from(provided, 'utf8');
    const b = Buffer.from(expected, 'utf8');
    return a.length === b.length && crypto.timingSafeEqual(a, b);
}

async function startWebhookServer(bot, { port, webhookUrl, secretToken, allowedUpdates }) {
    if (!secretToken || !SECRET_REGEX.test(secretToken)) {
        throw new Error('WEBHOOK_SECRET must be 32-256 chars matching [A-Za-z0-9_-]');
    }

    const app = express();
    app.disable('x-powered-by');
    app.use(express.json({ limit: '1mb' }));

    app.get('/health', (_req, res) => {
        res.status(200).json({ status: 'ok', uptime: Math.floor(process.uptime()) });
    });

    app.post('/webhook', async (req, res) => {
        const headerSecret = req.headers['x-telegram-bot-api-secret-token'];
        if (!isValidSecret(headerSecret, secretToken)) {
            return res.sendStatus(403);
        }

        try {
            await bot.handleUpdate(req.body, res);
            if (!res.headersSent) {
                res.sendStatus(200);
            }
        } catch (err) {
            console.error('Error handling webhook update:', err.message);
            if (!res.headersSent) {
                res.sendStatus(200); // Acknowledge to prevent infinite retry storms on app bugs
            }
        }
    });

    const normalizedUrl = webhookUrl.replace(/\/+$/, '');
    const fullWebhookUrl = `${normalizedUrl}/webhook`;

    await bot.telegram.setWebhook(fullWebhookUrl, {
        max_connections: 40,
        secret_token: secretToken,
        ...(allowedUpdates ? { allowed_updates: allowedUpdates } : {}),
    });

    const server = app.listen(port, () => {
        console.log(`Webhook server listening on port ${port}`);
    });

    return server;
}

module.exports = { startWebhookServer };
```

### Dual-Mode Startup (`bot.js`)
Switch automatically between polling and webhooks via `WEBHOOK_URL`:

```js
const webhookUrl = process.env.WEBHOOK_URL?.trim();
if (webhookUrl) {
    const port = Number(process.env.PORT || 3000);
    const secretToken = process.env.WEBHOOK_SECRET?.trim();
    httpServer = await startWebhookServer(bot, { port, webhookUrl, secretToken });
} else {
    await bot.launch();
}
```

On graceful shutdown (`SIGINT`/`SIGTERM`), close `httpServer?.close()` in webhook mode or call `bot.stop(signal)` in polling mode, then `await closeDatabase()`.

> **Preserve pending updates**: Do not call `deleteWebhook({ drop_pending_updates: true })` unless the user explicitly asks to discard queued updates.

---

## Serverless (Vercel)

```js
// api/webhook.js
const crypto = require('crypto');
const { Telegraf } = require('telegraf');

const bot = new Telegraf(process.env.TOKEN_BOT);
// Register handlers here...

module.exports = async function handler(req, res) {
    if (req.method !== 'POST') {
        return res.status(200).json({ status: 'ok' });
    }

    const expected = process.env.WEBHOOK_SECRET || '';
    const provided = req.headers['x-telegram-bot-api-secret-token'];
    if (
        !expected ||
        typeof provided !== 'string' ||
        provided.length !== expected.length ||
        !crypto.timingSafeEqual(Buffer.from(provided), Buffer.from(expected))
    ) {
        return res.status(403).send('Forbidden');
    }

    await bot.handleUpdate(req.body, res);
    if (!res.writableEnded) res.status(200).send('OK');
};
```

---

## Local Development Tunnels

Expose a local Express webhook server over HTTPS:
```bash
ngrok http 3000
# or
npx localtunnel --port 3000
```

Set `WEBHOOK_URL=https://<tunnel-domain>` and `WEBHOOK_SECRET=<random-32+-char-string>` in `.env`.

---

## Security & Troubleshooting

- **Telegram Webhook IP Ranges** (optional firewall allowlist): `149.154.160.0/20`, `91.108.4.0/22`.
- **Inspect webhook health** (never log the token):
  ```js
  const info = await bot.telegram.getWebhookInfo();
  console.log({ url: info.url, pending: info.pending_update_count, lastError: info.last_error_message });
  ```

| Symptom | Cause | Fix |
|---------|-------|-----|
| Webhook receives no updates | Wrong URL or invalid SSL cert | Check `getWebhookInfo()` (`last_error_message`) |
| `409 Conflict` | Polling and webhook both active | Stop polling or call `deleteWebhook` |
| `401 Unauthorized` | Invalid or revoked `TOKEN_BOT` | Verify with `bot.telegram.getMe()` |
| Duplicate updates | Endpoint did not return HTTP `200` | Ensure `res.sendStatus(200)` on all paths |
| Timeouts | Handler takes >60s | Offload heavy jobs async and ack `200` fast |

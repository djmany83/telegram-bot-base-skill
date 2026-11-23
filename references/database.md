# Database Reference (`database.js`)

## Option A: MySQL (Default)

Uses `mysql2/promise` with a lazy singleton pool, `utf8mb4` charset (full emoji support), optional TLS (`DB_SSL=true`), and support for either `DATABASE_URL` or individual `DB_*` variables.

```js
const mysql = require('mysql2/promise');

function buildPoolConfig() {
    const baseConfig = {
        waitForConnections: true,
        connectionLimit: 10,
        queueLimit: 0,
        enableKeepAlive: true,
        connectTimeout: 10000,
        charset: 'utf8mb4',
    };

    if (process.env.DB_SSL === 'true') {
        baseConfig.ssl = { rejectUnauthorized: true };
    }

    const databaseUrl = process.env.DATABASE_URL?.trim();
    if (databaseUrl) {
        const parsed = new URL(databaseUrl);
        return {
            ...baseConfig,
            host: parsed.hostname,
            port: parsed.port ? Number(parsed.port) : 3306,
            user: decodeURIComponent(parsed.username),
            password: decodeURIComponent(parsed.password),
            database: parsed.pathname.replace(/^\//, ''),
        };
    }

    return {
        ...baseConfig,
        host: process.env.DB_HOST || '127.0.0.1',
        port: Number(process.env.DB_PORT || 3306),
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        database: process.env.DB_NAME,
    };
}

let pool = null;

function getPool() {
    if (!pool) {
        pool = mysql.createPool(buildPoolConfig());
    }
    return pool;
}

async function run(sql, params = []) {
    const [result] = await getPool().execute(sql, params);
    return { lastID: result.insertId, changes: result.affectedRows };
}

async function get(sql, params = []) {
    const [rows] = await getPool().execute(sql, params);
    return rows[0] || null;
}

async function all(sql, params = []) {
    const [rows] = await getPool().execute(sql, params);
    return rows;
}

async function initDatabase() {
    const connection = await getPool().getConnection();
    connection.release();
    console.log('Conexión exitosa con la base de datos MySQL');

    await run(
        `CREATE TABLE IF NOT EXISTS users (
            id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
            user_id VARCHAR(64) NOT NULL UNIQUE,
            username VARCHAR(255),
            first_name VARCHAR(255),
            type VARCHAR(32),
            title_group VARCHAR(255),
            is_active TINYINT(1) NOT NULL DEFAULT 1,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            INDEX idx_users_active_type (is_active, type)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci`
    );

    // Idempotent column check for upgrades from older schemas
    const activeColumn = await get(
        `SELECT COLUMN_NAME
         FROM information_schema.COLUMNS
         WHERE TABLE_SCHEMA = DATABASE()
           AND TABLE_NAME = 'users'
           AND COLUMN_NAME = 'is_active'`
    );
    if (!activeColumn) {
        await run('ALTER TABLE users ADD COLUMN is_active TINYINT(1) NOT NULL DEFAULT 1');
    }

    // Add bot-specific tables here using ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
}

async function closeDatabase() {
    if (pool) {
        await pool.end();
        pool = null;
    }
}

module.exports = { getPool, run, get, all, initDatabase, closeDatabase };
```

### MySQL Notes
- Always use `pool.execute(sql, params)` for prepared statements with `?` placeholders.
- For `LIMIT` and `OFFSET` in `mysql2` prepared statements, validate as safe integers (`Number.isSafeInteger` / clamped bounds) or pass via `pool.query` if dynamic binding fails on older MySQL versions.
- Always store Telegram IDs (`user_id`, `chat_id`) as `VARCHAR(64)` (or `BIGINT`) because Telegram group/supergroup IDs can exceed 32-bit signed integers.

---

## Option B: SQLite (Lightweight Alternative)

Use only when the user explicitly wants a zero-server/local bot. Expose the **exact same async interface** (`run`, `get`, `all`, `initDatabase`, `closeDatabase`) so middlewares and admin handlers work without changes.

```js
const fs = require('fs');
const path = require('path');
const Database = require('better-sqlite3');

let db = null;

function getDb() {
    if (!db) {
        const dbPath = path.resolve(process.env.DB_PATH || './db/bot.db');
        fs.mkdirSync(path.dirname(dbPath), { recursive: true });
        db = new Database(dbPath);
        db.pragma('journal_mode = WAL');
        db.pragma('foreign_keys = ON');
    }
    return db;
}

async function run(sql, params = []) {
    const info = getDb().prepare(sql).run(...params);
    return { lastID: Number(info.lastInsertRowid), changes: info.changes };
}

async function get(sql, params = []) {
    return getDb().prepare(sql).get(...params) || null;
}

async function all(sql, params = []) {
    return getDb().prepare(sql).all(...params);
}

async function initDatabase() {
    getDb();
    console.log('Conexión exitosa con la base de datos SQLite');

    await run(
        `CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT NOT NULL UNIQUE,
            username TEXT,
            first_name TEXT,
            type TEXT,
            title_group TEXT,
            is_active INTEGER NOT NULL DEFAULT 1,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )`
    );
    await run('CREATE INDEX IF NOT EXISTS idx_users_active_type ON users(is_active, type)');

    const columns = await all('PRAGMA table_info(users)');
    if (!columns.some((col) => col.name === 'is_active')) {
        await run('ALTER TABLE users ADD COLUMN is_active INTEGER NOT NULL DEFAULT 1');
    }
}

async function closeDatabase() {
    if (db) {
        db.close();
        db = null;
    }
}

module.exports = { getDb, run, get, all, initDatabase, closeDatabase };
```

### SQLite Notes
- Store the database file in `db/bot.db` (configured via `DB_PATH` env var) and mount a Docker volume for `/app/db`.
- Adjust MySQL-specific syntax in `userTracking.js`:
  - Replace `ON DUPLICATE KEY UPDATE ... VALUES(col)` with `ON CONFLICT(user_id) DO UPDATE SET col = excluded.col`.
- In `/db` backup, either export JSON (consistent with MySQL) or run `getDb().pragma('wal_checkpoint(TRUNCATE)')` before sending the `.db` file.

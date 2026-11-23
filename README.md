# Telegram Bot Base — Agent Skill

[![Node.js](https://img.shields.io/badge/Node.js-%3E%3D22.0.0-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![Telegraf](https://img.shields.io/badge/Telegraf-v4-26A5E4?logo=telegram&logoColor=white)](https://telegraf.js.org/)
[![MySQL](https://img.shields.io/badge/MySQL-8%2B%20%7C%20SQLite-4479A1?logo=mysql&logoColor=white)](https://www.mysql.com/)
[![Docker](https://img.shields.io/badge/Docker-Non--Root%20Ready-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)

Skill reutilizable para agentes de IA que proporciona una arquitectura modular y lista para producción para crear o mejorar **bots de Telegram** con **Node.js**, **Telegraf v4**, **MySQL** (o **SQLite** opcional) y **Docker**.

---

## ¿Qué incluye esta skill?

Al activar esta skill, el agente sigue una guía estandarizada que cubre:

- **Arquitectura modular limpia**: separación clara entre punto de entrada (`bot.js`), capa de base de datos, middlewares, handlers públicos/admin y utilidades.
- **Capa de base de datos unificada (`database.js`)**:
  - **MySQL por defecto** (`mysql2/promise`) con pool de conexiones, soporte `utf8mb4` (emojis), TLS opcional (`DB_SSL=true`), `DATABASE_URL` o variables individuales `DB_*` e índice compuesto `(is_active, type)`.
  - **SQLite opcional** (`better-sqlite3` con WAL y `foreign_keys`) exponiendo exactamente la misma interfaz asíncrona (`run`, `get`, `all`, `initDatabase`, `closeDatabase`).
- **Tracking automático de usuarios y grupos**:
  - Middleware que registra y actualiza chats `private`, `group` y `supergroup` sin bloquear el pipeline ante errores de BD.
  - Listener `my_chat_member` que detecta altas, bloqueos o expulsiones (`kicked`/`left`) y actualiza `is_active` automáticamente.
- **Panel de administración privado y multi-admin**:
  - Autorización estricta por `ADMIN_ID` (soporta múltiples IDs separados por coma) restringida **exclusivamente a chats privados**.
  - Menú de comandos de admin registrado por `chat_id` (`setMyCommands` con `scope: { type: 'chat' }`) para que los usuarios normales nunca vean comandos internos.
  - Comandos y botones incluidos de serie:
    - `/admin` (`🛠️ Panel Admin`) — Acceso al teclado de administración.
    - `/stats` (`📊 Estadísticas`) — Métricas de usuarios activos/inactivos (privados vs grupos) y uptime.
    - `/members` (`👥 Usuarios`) — Listado paginado con botones inline (`private`, `group`, `inactive`).
    - `/status` (`🩺 Estado`) — Latencia en vivo de la BD (`SELECT 1`), memoria RSS, uptime y versión de Node.
    - `/db` (`💾 Respaldo BD`) — Exportación completa de tablas en JSON fechada enviada como documento.
    - `/msg <texto>` (`📢 Difundir`) — Difusión controlada a chats activos con retardo de `50ms`, manejo de `429 Too Many Requests` (`retry_after`) y baja automática ante errores permanentes (`403`, bloqueado, expulsado, desactivado).
- **Hardening de seguridad y despliegue**:
  - Consultas parametrizadas (`?`), fail-fast si falta `TOKEN_BOT`, manejador global `bot.catch` y apagado limpio en `SIGINT`/`SIGTERM`.
  - Plantillas de `Dockerfile` (usuario no-root `node:22-slim`), `.dockerignore`, `docker-compose.yml` (con `--dns-result-order=ipv4first` para evitar timeouts IPv6 en Docker) y `.env.example`.

---

## Estructura del repositorio

```text
telegram-bot-base-skill/
├── SKILL.md                             # Instrucciones principales, flujo de trabajo y checklist de validación
├── README.md                            # Documentación de la skill
└── references/
    ├── database.md                      # Implementación de referencia para MySQL y SQLite
    ├── admin-and-tracking.md            # Auth multi-admin, middleware de tracking y panel admin
    └── security-and-deploy.md           # Checklist de seguridad, Docker, .env.example y plantilla bot.js
```

---

## Estructura del proyecto que genera

```text
<bot-project>/
├── bot.js                      # Punto de entrada, ciclo de vida y apagado limpio
├── database.js                 # Pool/cliente BD, run/get/all, initDatabase, closeDatabase
├── middlewares/
│   └── userTracking.js         # Registro automático de usuarios/grupos + my_chat_member
├── handlers/
│   ├── commands.js             # Comandos públicos + USER_COMMANDS_MENU
│   ├── admin.js                # Panel admin, paginación inline, respaldo JSON y difusión
│   └── hears.js                # Handlers de botones de teclado de respuesta
├── utils/
│   ├── auth.js                 # Validación multi-admin solo en chat privado
│   └── keyboard.js             # Teclados principal, admin y cancelar
├── .env.example                # Variables de entorno documentadas
├── .gitignore
├── .dockerignore
├── Dockerfile
├── docker-compose.yml
└── package.json                # Scripts: start, dev (--watch) y check (node --check)
```

---

## Instalación

Esta skill sigue el estándar abierto `SKILL.md`, por lo que es compatible con **Antigravity** y otros agentes que soporten skills basadas en carpetas.

### Opción 1: Instalación global en Antigravity

Clona el repositorio en tu carpeta global de skills:

```bash
git clone https://github.com/djmany83/telegram-bot-base-skill.git ~/.gemini/config/skills/telegram-bot-base
```

En Windows (PowerShell):

```powershell
git clone https://github.com/djmany83/telegram-bot-base-skill.git "$env:USERPROFILE\.gemini\config\skills\telegram-bot-base"
```

### Opción 2: Instalación por proyecto

Clónala dentro del directorio de skills de tu repositorio:

```bash
git clone https://github.com/djmany83/telegram-bot-base-skill.git .agents/skills/telegram-bot-base
```

### Actualizar la skill

```bash
git -C ~/.gemini/config/skills/telegram-bot-base pull
```

---

## Ejemplos de uso

Una vez instalada, el agente cargará la skill automáticamente cuando le pidas crear o mejorar un bot de Telegram. Por ejemplo:

- **Crear un bot nuevo con MySQL (por defecto):**
  > "Crea un bot de Telegram para gestionar recordatorios diarios usando la skill `telegram-bot-base`."
- **Crear un bot ligero con SQLite:**
  > "Crea un bot de Telegram de notas rápidas con SQLite en lugar de MySQL."
- **Mejorar un bot existente:**
  > "Añade a este bot de Telegram el sistema de tracking de usuarios/grupos, panel admin privado con `/stats`, `/members`, `/db`, `/msg` y configuración Docker."

---

## Requisitos del stack generado

- **Node.js** `>= 22.0.0`
- **MySQL 8+** (o **SQLite** si se solicita la variante ligera)
- Token de bot obtenido en [@BotFather](https://t.me/BotFather)
- **Docker** y **Docker Compose** (opcional, para despliegue en contenedor)

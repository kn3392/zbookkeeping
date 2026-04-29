# 📒 ezBookkeeping — Complete System Guide

> A lightweight, self-hosted personal finance application built with **Go (backend)** + **Vue 3 (frontend)**.  
> This guide covers: system architecture, data flow, local setup, and full deployment to **Render + Supabase**.

---

## 📑 Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [System Architecture & Design](#3-system-architecture--design)
4. [Directory Structure](#4-directory-structure)
5. [Data Models](#5-data-models)
6. [API & Data Flow](#6-api--data-flow)
7. [Configuration Reference](#7-configuration-reference)
8. [Local Development Setup](#8-local-development-setup)
9. [🚀 Deploy to Render + Supabase (Free)](#9-deploy-to-render--supabase-free)
10. [Environment Variables Reference](#10-environment-variables-reference)
11. [Features Overview](#11-features-overview)

---

## 1. Project Overview

**ezBookkeeping** is a self-hosted personal bookkeeping system. It provides:

- Income / Expense / Transfer transaction tracking
- Multi-account, multi-currency support with live exchange rates
- Transaction categories, tags, templates, and scheduled transactions
- Mobile PWA + Desktop web interfaces
- AI receipt image recognition (optional LLM integration)
- OAuth 2.0 / OIDC login support
- MCP (Model Context Protocol) server for AI/LLM access
- Data import/export (CSV, TSV, Excel)

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| **Backend Language** | Go 1.26 |
| **Web Framework** | Gin (gin-gonic/gin) |
| **ORM / DB Layer** | xorm |
| **Databases Supported** | SQLite3, MySQL, PostgreSQL |
| **Frontend Framework** | Vue 3 + TypeScript |
| **UI Components** | Vuetify 3 (Desktop), Framework7 (Mobile) |
| **State Management** | Pinia |
| **Charts** | ECharts + vue-echarts |
| **Maps** | Leaflet.js |
| **Build Tool** | Vite 7 |
| **Auth** | JWT (golang-jwt) + OAuth2 / OIDC |
| **Scheduler** | gocron v2 |
| **Object Storage** | Local Filesystem / MinIO / WebDAV |
| **Email** | SMTP (gopkg.in/mail.v2) |
| **Containerization** | Docker (multi-stage build) |
| **Deployment** | Render (Docker runtime) |
| **Database Hosting** | Supabase (PostgreSQL) |

---

## 3. System Architecture & Design

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│                                                                 │
│  ┌──────────────────┐          ┌──────────────────┐            │
│  │   Mobile PWA     │          │   Desktop Web    │            │
│  │  /mobile         │          │  /desktop or /   │            │
│  │  (Framework7)    │          │  (Vuetify 3)     │            │
│  └────────┬─────────┘          └────────┬─────────┘            │
│           │  Vue 3 + Pinia + Axios       │                      │
└───────────┼─────────────────────────────┼──────────────────────┘
            │        HTTPS / HTTP          │
            ▼                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     GO WEB SERVER (Gin)                         │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Static File │  │  Middleware  │  │   Route Groups       │  │
│  │  Serving    │  │  - RequestID │  │   /api/v1/...        │  │
│  │  /js /css   │  │  - JWT Auth  │  │   /oauth2/...        │  │
│  │  /img       │  │  - Request   │  │   /mcp               │  │
│  └─────────────┘  │    Logging   │  │   /proxy/...         │  │
│                   │  - Recovery  │  │   /avatar            │  │
│                   │  - GZip      │  │   /pictures          │  │
│                   └──────────────┘  └──────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    API HANDLERS (pkg/api)                │  │
│  │  accounts | transactions | categories | tags | users     │  │
│  │  tokens | exchange_rates | data_mgmt | llm | mcp        │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                           │
│  ┌──────────────────▼───────────────────────────────────────┐  │
│  │                 SERVICES LAYER (pkg/services)            │  │
│  │  Business logic, validation, orchestration               │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                           │
│  ┌──────────────────▼───────────────────────────────────────┐  │
│  │              DATASTORE LAYER (pkg/datastore)             │  │
│  │  xorm ORM — MySQL | PostgreSQL | SQLite3                 │  │
│  └──────────────────┬───────────────────────────────────────┘  │
└─────────────────────┼───────────────────────────────────────────┘
                      │
          ┌───────────▼───────────┐
          │  DATABASE (Supabase)  │
          │   PostgreSQL          │
          └───────────────────────┘

BACKGROUND SERVICES:
┌──────────────────────────────┐   ┌─────────────────────────┐
│    CRON SCHEDULER (gocron)   │   │  EXCHANGE RATES SERVICE  │
│  - Clean expired tokens      │   │  Euro Central Bank etc.  │
│  - Create scheduled txns     │   └─────────────────────────┘
└──────────────────────────────┘
┌──────────────────────────────┐   ┌─────────────────────────┐
│    OBJECT STORAGE            │   │  LLM / AI SERVICE        │
│  Local / MinIO / WebDAV      │   │  OpenAI / Anthropic etc. │
│  (avatars, pictures)         │   └─────────────────────────┘
└──────────────────────────────┘
```

---

## 4. Directory Structure

```
ezbookkeeping/
│
├── ezbookkeeping.go          # Main entry point — CLI app bootstrap
├── go.mod / go.sum           # Go module dependencies
├── package.json              # Node/frontend dependencies
├── vite.config.ts            # Vite build configuration
├── Dockerfile                # Multi-stage Docker build
├── render.yaml               # Render.com deployment config
├── build.sh / build.bat      # Build scripts
│
├── cmd/                      # CLI command handlers
│   ├── webserver.go          # Web server startup + route registration
│   ├── database.go           # DB migration commands
│   ├── initializer.go        # System initialization
│   ├── user_data.go          # User data management CLI
│   └── cron_jobs.go          # Cron job CLI commands
│
├── conf/
│   └── ezbookkeeping.ini     # Master configuration file
│
├── pkg/                      # Core Go packages
│   ├── api/                  # HTTP handler layer
│   │   ├── accounts.go
│   │   ├── transactions.go
│   │   ├── authorizations.go
│   │   ├── users.go
│   │   ├── tokens.go
│   │   ├── exchange_rates.go
│   │   ├── large_language_models.go
│   │   ├── model_context_protocols.go
│   │   └── ... (28 files)
│   │
│   ├── services/             # Business logic layer
│   │   ├── accounts.go
│   │   ├── transactions.go
│   │   ├── users.go
│   │   ├── tokens.go
│   │   └── ... (19 files)
│   │
│   ├── models/               # Data models / DTOs
│   │   ├── user.go
│   │   ├── transaction.go
│   │   ├── account.go
│   │   └── ... (35 files)
│   │
│   ├── datastore/            # Database abstraction (xorm)
│   ├── settings/             # Config loading & validation
│   ├── middlewares/          # JWT auth, rate limit, logging
│   ├── auth/oauth2/          # OAuth2 / OIDC provider
│   ├── exchangerates/        # 15+ exchange rate data sources
│   ├── storage/              # Local / MinIO / WebDAV backends
│   ├── cron/                 # Scheduled job scheduler
│   ├── llm/                  # AI/LLM provider integrations
│   ├── mcp/                  # Model Context Protocol server
│   ├── mail/                 # SMTP email service
│   ├── log/                  # Structured logging (logrus)
│   ├── errs/                 # Error definitions
│   └── utils/                # Shared utilities
│
└── src/                      # Frontend (Vue 3 + TypeScript)
    ├── mobile.html           # Mobile PWA entry HTML
    ├── desktop.html          # Desktop entry HTML
    ├── mobile-main.ts        # Mobile app bootstrap
    ├── desktop-main.ts       # Desktop app bootstrap
    ├── MobileApp.vue         # Mobile root component
    ├── DesktopApp.vue        # Desktop root component
    │
    ├── views/
    │   ├── mobile/           # Mobile-specific page components
    │   │   ├── HomePage.vue
    │   │   ├── LoginPage.vue
    │   │   ├── SignupPage.vue
    │   │   ├── SettingsPage.vue
    │   │   ├── transactions/
    │   │   ├── accounts/
    │   │   ├── categories/
    │   │   ├── tags/
    │   │   ├── statistics/
    │   │   └── ...
    │   └── desktop/          # Desktop-specific pages
    │
    ├── stores/               # Pinia state stores
    │   ├── account.ts
    │   ├── transaction.ts
    │   ├── statistics.ts
    │   ├── exchangeRates.ts
    │   ├── user.ts
    │   └── ... (18 stores)
    │
    ├── router/               # Vue Router configuration
    ├── locales/              # i18n translation files
    ├── components/           # Reusable UI components
    └── lib/                  # Frontend utilities
```

---

## 5. Data Models

### Core Entities

| Model | Key Fields |
|---|---|
| **User** | uid, username, email, password_hash, nickname, avatar, default_currency, language, timezone |
| **Account** | id, uid, name, type (cash/card/etc.), currency, balance, icon, color, hidden |
| **Transaction** | id, uid, type (income/expense/transfer), time, amount, currency, account_id, category_id, comment, geo_location |
| **TransactionCategory** | id, uid, name, type, parent_id, icon, color, hidden |
| **TransactionTag** | id, uid, name, hidden |
| **TransactionTagGroup** | id, uid, name |
| **TransactionTemplate** | id, uid, name, type, amount, account_id, category_id (for scheduled txns) |
| **Token** | id, uid, token_type, secret, created_ip, expired_time |
| **TwoFactor** | id, uid, secret (TOTP) |

### Transaction Types
```
0 = Modify Balance
1 = Income
2 = Expense
3 = Transfer
```

---

## 6. API & Data Flow

### Authentication Flow

```
User Login Request
      │
      ▼
POST /api/authorize.json
      │
      ▼
[Middleware: RequestID, RequestLog]
      │
      ▼
api.Authorizations.AuthorizeHandler
      │ validates username + password
      ▼
services.Tokens → generates JWT
      │
      ├── If 2FA enabled → returns temp token
      │   └── POST /api/2fa/authorize.json → full JWT
      │
      └── Returns JWT token to client
            │
            ▼
    Client stores JWT
    All subsequent calls:
    GET/POST /api/v1/... (Authorization: Bearer <token>)
```

### Transaction Create Flow

```
POST /api/v1/transactions/add.json
      │
      ▼
[JWT Middleware validates token]
      │
      ▼
api.Transactions.TransactionCreateHandler
      │ parses & validates request body
      ▼
services.Transactions.CreateTransaction()
      │ ├── validates account ownership
      │ ├── validates category
      │ ├── handles currency conversion
      │ ├── updates account balance
      │ └── saves transaction + tags
      ▼
datastore (xorm → PostgreSQL/MySQL/SQLite)
      │
      ▼
JSON response → Frontend Pinia store updated
      │
      ▼
UI re-renders (Vue 3 reactivity)
```

### Exchange Rate Flow

```
[Cron Job - daily]
      │
      ▼
exchangerates.DataProvider.GetLatestExchangeRates()
  (Euro Central Bank / Bank of Canada / etc.)
      │
      ▼
Cached in memory
      │
      ▼
GET /api/v1/exchange_rates/latest.json
      │
      ▼
Frontend converts amounts on the fly
```

### Frontend → Backend Data Flow

```
Vue Component (user action)
      │
      ▼
Pinia Store action (e.g. transactionStore.createTransaction)
      │
      ▼
axios HTTP call → /api/v1/...
  (with JWT Bearer token in headers)
      │
      ▼
Gin Router → Middleware → API Handler
      │
      ▼
Service Layer (business logic)
      │
      ▼
xorm ORM → Database
      │
      ▼
Response JSON → Pinia store state updated
      │
      ▼
Vue 3 reactivity → UI updates automatically
```

---

## 7. Configuration Reference

The main config file is `conf/ezbookkeeping.ini`. All settings can be overridden with **environment variables** using the prefix `EBK_`:

```ini
[global]
mode = production           # or development

[server]
http_addr = 0.0.0.0
http_port = 8080
domain = localhost
root_url = %(protocol)s://%(domain)s:%(http_port)s/

[database]
type = postgres             # mysql | postgres | sqlite3
host = 127.0.0.1:5432
name = ezbookkeeping
user = postgres
passwd = yourpassword
ssl_mode = require          # disable | require | verify-full

[security]
secret_key = CHANGE_THIS    # Required! Used for JWT signing

[auth]
enable_register = true
enable_two_factor = true
enable_oauth2_auth = false

[storage]
type = local_filesystem     # local_filesystem | minio | webdav
local_filesystem_path = storage/

[exchange_rates]
data_source = euro_central_bank
```

### Environment Variable Mapping

Config keys map to env vars as:
`[section]_key` → `EBK_SECTION_KEY` (uppercase, underscores)

Examples:
```
EBK_GLOBAL_MODE          → [global] mode
EBK_SERVER_HTTP_PORT     → [server] http_port
EBK_DATABASE_TYPE        → [database] type
EBK_DATABASE_HOST        → [database] host
EBK_DATABASE_PASSWD      → [database] passwd
EBK_SECURITY_SECRET_KEY  → [security] secret_key
```

---

## 8. Local Development Setup

### Prerequisites

- Go 1.22+
- Node.js 20+
- Git

### Step 1: Clone and Install

```bash
git clone https://github.com/mayswind/ezbookkeeping.git
cd ezbookkeeping
npm install
```

### Step 2: Configure (SQLite for local dev)

Edit `conf/ezbookkeeping.ini`:
```ini
[database]
type = sqlite3
db_path = data/ezbookkeeping.db

[security]
secret_key = my-local-secret-key-change-me
```

### Step 3: Build Frontend

```bash
npm run build
```

### Step 4: Build & Run Backend

```bash
go run ezbookkeeping.go server run
```

App runs at: **http://localhost:8080**

- Desktop UI: `http://localhost:8080/` or `/desktop`
- Mobile PWA: `http://localhost:8080/mobile`
- Health check: `http://localhost:8080/healthz.json`

### Build Scripts

```bash
# Build backend only
./build.sh backend

# Build frontend only
./build.sh frontend

# Build everything
./build.sh

# Windows
./build.bat
```

---

## 9. 🚀 Deploy to Render + Supabase (Free)

This is the recommended **free-tier** deployment using:
- **Render.com** → hosts the Go application as a Docker container
- **Supabase** → provides managed PostgreSQL database

---

### PART A: Set Up Supabase Database

#### Step 1: Create a Supabase Account
1. Go to [https://supabase.com](https://supabase.com)
2. Click **Start your project** → Sign up (free)

#### Step 2: Create a New Project
1. Click **New project**
2. Fill in:
   - **Name**: `ezbookkeeping`
   - **Database Password**: Set a strong password (**save it!**)
   - **Region**: Choose closest to you
3. Click **Create new project** (takes ~2 minutes)

#### Step 3: Get Your Database Connection Details
1. In your project dashboard, go to **Settings** → **Database**
2. Scroll to **Connection parameters** section
3. Copy these values:
   ```
   Host:     db.xxxxxxxxxxxxxxxxxxxx.supabase.co
   Port:     5432
   Database: postgres
   User:     postgres
   Password: [your password from Step 2]
   ```

> **Important:** Use the **direct connection** host (not the pooler) for the Go app.

#### Step 4: Note SSL Requirement
Supabase requires SSL. The SSL mode should be `require`.

---

### PART B: Deploy to Render

#### Step 1: Push Code to GitHub

```bash
# In your project directory
git init
git add .
git commit -m "Initial commit"

# Create a new repo on github.com, then:
git remote add origin https://github.com/YOUR_USERNAME/ezbookkeeping.git
git branch -M main
git push -u origin main
```

#### Step 2: Create Render Account
1. Go to [https://render.com](https://render.com)
2. Sign up with GitHub (recommended for easy repo access)

#### Step 3: Create a New Web Service
1. Click **New +** → **Web Service**
2. Connect your GitHub account if not done
3. Select your `ezbookkeeping` repository
4. Click **Connect**

#### Step 4: Configure the Service

Fill in the following:

| Field | Value |
|---|---|
| **Name** | `ezbookkeeping` |
| **Region** | Choose closest to you |
| **Branch** | `main` |
| **Runtime** | `Docker` |
| **Plan** | `Free` |

Render auto-detects the `Dockerfile` in your repo.

#### Step 5: Set Environment Variables

In the Render dashboard, under **Environment**, add these variables:

| Key | Value |
|---|---|
| `EBK_GLOBAL_MODE` | `production` |
| `EBK_SERVER_HTTP_PORT` | `8080` |
| `EBK_DATABASE_TYPE` | `postgres` |
| `EBK_DATABASE_HOST` | `db.xxxx.supabase.co:5432` |
| `EBK_DATABASE_NAME` | `postgres` |
| `EBK_DATABASE_USER` | `postgres` |
| `EBK_DATABASE_PASSWD` | `your_supabase_db_password` |
| `EBK_DATABASE_SSL_MODE` | `require` |
| `EBK_SECURITY_SECRET_KEY` | `generate-a-random-string-here` |

> **Tip for SECRET_KEY:** Run this to generate one:
> ```bash
> # On Linux/Mac:
> openssl rand -hex 32
> # On Windows PowerShell:
> -join ((65..90)+(97..122)+(48..57) | Get-Random -Count 32 | % {[char]$_})
> ```

#### Step 6: Deploy
1. Click **Create Web Service**
2. Render will build the Docker image and deploy (~5-10 minutes)
3. Watch the logs — you should see:
   ```
   [webserver.startWebServer] will run at http://0.0.0.0:8080
   ```
4. Your app URL will be: `https://ezbookkeeping-xxxx.onrender.com`

#### Step 7: First-Time Setup
1. Visit your Render URL
2. Register your first account (this becomes the admin)
3. Go to **Settings** and configure your preferred currency

---

### PART C: Configure render.yaml (Auto-Deploy)

Your repo already has a `render.yaml`. Update it to match:

```yaml
services:
  - type: web
    name: ezbookkeeping
    runtime: docker
    plan: free
    envVars:
      - key: EBK_GLOBAL_MODE
        value: production
      - key: EBK_SERVER_HTTP_PORT
        value: 8080
      - key: EBK_DATABASE_TYPE
        value: postgres
      - key: EBK_DATABASE_HOST
        sync: false          # Set manually in Render dashboard
      - key: EBK_DATABASE_NAME
        value: postgres
      - key: EBK_DATABASE_USER
        value: postgres
      - key: EBK_DATABASE_PASSWD
        sync: false          # Set manually in Render dashboard
      - key: EBK_DATABASE_SSL_MODE
        value: require
      - key: EBK_SECURITY_SECRET_KEY
        generateValue: true  # Render auto-generates this
```

---

### PART D: Troubleshooting Deployment

#### Database Connection Refused
```
Error: dial tcp: connect: connection refused
```
**Fix:** Make sure your `EBK_DATABASE_HOST` includes the port:
```
db.xxxxxxxxxxxx.supabase.co:5432
```

#### SSL Required Error
```
Error: pq: SSL is not enabled on the server
```
**Fix:** Set `EBK_DATABASE_SSL_MODE=require`

#### Service Sleeping (Free Tier)
Render free services sleep after 15 minutes of inactivity.
- Use [UptimeRobot](https://uptimerobot.com) (free) to ping your `/healthz.json` every 10 minutes to keep it awake.
- URL to monitor: `https://your-app.onrender.com/healthz.json`

#### Build Fails: Go Version
If build fails due to Go version mismatch, check your `Dockerfile` — it uses `golang:1.26.2-alpine3.23`.

---

## 10. Environment Variables Reference

All environment variables use prefix `EBK_` followed by section and key in UPPERCASE:

```bash
# Global
EBK_GLOBAL_MODE=production

# Server
EBK_SERVER_HTTP_PORT=8080
EBK_SERVER_DOMAIN=your-app.onrender.com
EBK_SERVER_ROOT_URL=https://your-app.onrender.com/

# Database (Supabase PostgreSQL)
EBK_DATABASE_TYPE=postgres
EBK_DATABASE_HOST=db.xxxx.supabase.co:5432
EBK_DATABASE_NAME=postgres
EBK_DATABASE_USER=postgres
EBK_DATABASE_PASSWD=your_password
EBK_DATABASE_SSL_MODE=require
EBK_DATABASE_AUTO_UPDATE_DATABASE=true

# Security
EBK_SECURITY_SECRET_KEY=your_random_secret_key
EBK_SECURITY_TOKEN_EXPIRED_TIME=2592000

# Auth
EBK_AUTH_ENABLE_REGISTER=true
EBK_AUTH_ENABLE_TWO_FACTOR=true

# User Features
EBK_USER_ENABLE_TRANSACTION_PICTURE=true
EBK_USER_ENABLE_SCHEDULED_TRANSACTION=true

# Data
EBK_DATA_ENABLE_EXPORT=true
EBK_DATA_ENABLE_IMPORT=true

# Storage (local by default on Render)
EBK_STORAGE_TYPE=local_filesystem

# Logging
EBK_LOG_MODE=console
EBK_LOG_LEVEL=info

# Exchange Rates
EBK_EXCHANGE_RATES_DATA_SOURCE=euro_central_bank
```

---

## 11. Features Overview

### Financial Management
- ✅ Income, Expense, Transfer transactions
- ✅ Multi-account (Cash, Bank, Credit Card, etc.)
- ✅ Multi-currency with real-time exchange rates
- ✅ Transaction categories (hierarchical)
- ✅ Transaction tags + tag groups
- ✅ Transaction templates
- ✅ Scheduled/recurring transactions
- ✅ Transaction pictures/receipts
- ✅ AI receipt image recognition (OpenAI, Anthropic, Ollama, etc.)

### Analytics
- ✅ Spending statistics by category, account, time
- ✅ Trend charts
- ✅ Asset tracking trends
- ✅ Reconciliation statements
- ✅ Custom Insights Explorer

### User Interface
- ✅ Mobile-optimized PWA (`/mobile`)
- ✅ Desktop web app (`/desktop`)
- ✅ Multi-language support (i18n)
- ✅ Multiple time zones
- ✅ Dark/Light theme
- ✅ Map integration for geo-tagged transactions

### Security & Auth
- ✅ Internal username/password auth
- ✅ Two-Factor Authentication (TOTP)
- ✅ OAuth 2.0 / OIDC (GitHub, Gitea, Nextcloud, custom)
- ✅ JWT-based session tokens
- ✅ API tokens
- ✅ Rate limiting per IP and per user

### Data Management
- ✅ Export to CSV / TSV
- ✅ Import from CSV, Excel (XLS/XLSX), OFX, QIF, and more
- ✅ Clear all data or per-account data
- ✅ MCP (AI/LLM access via Model Context Protocol)

### Infrastructure
- ✅ SQLite / MySQL / PostgreSQL support
- ✅ Docker deployment (multi-stage build)
- ✅ Local / MinIO / WebDAV object storage
- ✅ SMTP email (password reset, verification)
- ✅ Configurable via INI file or environment variables
- ✅ Auto database migration on startup
- ✅ Health check endpoint `/healthz.json`

---

## Quick Links

| Resource | URL |
|---|---|
| Health Check | `https://your-app.onrender.com/healthz.json` |
| Mobile UI | `https://your-app.onrender.com/mobile` |
| Desktop UI | `https://your-app.onrender.com/desktop` |
| API Base | `https://your-app.onrender.com/api/v1/` |
| Supabase Dashboard | `https://app.supabase.com` |
| Render Dashboard | `https://dashboard.render.com` |

---

*Generated from codebase analysis of ezBookkeeping v1.5.0*

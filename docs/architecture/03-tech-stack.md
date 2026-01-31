# Tech Stack & Deployment

**Language:** English | [中文](03-tech-stack.zh-CN.md)

Complete technology stack and deployment architecture documentation.

---

## Technology Stack

### Backend

| Category | Technology | Version | Purpose |
|----------|------------|---------|---------|
| **Language** | Go | 1.25.3 | Core backend |
| **Web Framework** | Gin | latest | HTTP routing, middleware |
| **ORM** | GORM | v2 | Database abstraction |
| **Database** | SQLite | 3.x | Default storage |
| **Database** | PostgreSQL | 15+ | Production option |
| **Auth** | golang-jwt | v5 | JWT tokens |
| **2FA** | pquerna/otp | latest | TOTP implementation |
| **WebSocket** | gorilla/websocket | latest | Real-time data |
| **Logging** | zerolog | latest | Structured logging |

### Frontend

| Category | Technology | Version | Purpose |
|----------|------------|---------|---------|
| **Framework** | React | 18.3.1 | UI library |
| **Language** | TypeScript | 5.8 | Type safety |
| **Build Tool** | Vite | 6.0.7 | Fast bundling |
| **CSS** | TailwindCSS | 3.4.17 | Utility-first styling |
| **State** | Zustand | 5.0.2 | State management |
| **Data Fetching** | SWR | 2.2.5 | Cache + revalidation |
| **HTTP Client** | Axios | 1.13.2 | API requests |
| **Charts** | Recharts | 2.15.2 | Data visualization |
| **Trading Charts** | lightweight-charts | 5.1.0 | Candlestick charts |
| **Animation** | Framer Motion | 12.23.24 | UI animations |
| **UI Components** | Radix UI | latest | Accessible primitives |
| **Icons** | Lucide React | latest | Icon library |

### Exchange SDKs

| Exchange | SDK | Type |
|----------|-----|------|
| Binance | `adshao/go-binance/v2` | Official |
| Bybit | `bybit-exchange/bybit.go.api` | Official |
| OKX | Custom SDK | In-house |
| Bitget | Custom SDK | In-house |
| Hyperliquid | Custom SDK | In-house |
| Aster | Custom SDK | In-house |
| Lighter | Custom SDK | In-house |

### AI Integrations

| Provider | Models | Integration |
|----------|--------|-------------|
| DeepSeek | V3, R1 | HTTP API |
| Alibaba Cloud | Qwen-Max, Qwen-Plus | HTTP API |
| OpenAI | GPT-4o, o1 | HTTP API |
| Anthropic | Claude Sonnet, Opus | HTTP API |
| Google | Gemini Pro, Flash | HTTP API |
| xAI | Grok | HTTP API |
| Moonshot | Kimi | HTTP API |

---

## Configuration

### Environment Variables

```bash
# ============================================
# Core Configuration
# ============================================

# JWT authentication secret (required)
JWT_SECRET=your-secret-key-here

# Server port (default: 8080)
API_SERVER_PORT=8080

# ============================================
# Database Configuration
# ============================================

# Database type: "sqlite" or "postgres"
DB_TYPE=sqlite

# SQLite path (when DB_TYPE=sqlite)
DB_PATH=data/data.db

# PostgreSQL config (when DB_TYPE=postgres)
DB_HOST=localhost
DB_PORT=5432
DB_USER=nofx
DB_PASSWORD=your-password
DB_NAME=nofx
DB_SSL_MODE=disable

# ============================================
# Encryption (Optional but Recommended)
# ============================================

# AES-256 encryption key for sensitive data (base64 encoded)
DATA_ENCRYPTION_KEY=your-base64-encoded-32-byte-key

# RSA private key for client-side encryption (PEM format)
RSA_PRIVATE_KEY=-----BEGIN PRIVATE KEY-----...

# Enable transport encryption (browser-side)
TRANSPORT_ENCRYPTION=false

# ============================================
# AI Model API Keys
# ============================================

DEEPSEEK_API_KEY=sk-xxx
QWEN_API_KEY=sk-xxx
OPENAI_API_KEY=sk-xxx
ANTHROPIC_API_KEY=sk-xxx
GOOGLE_API_KEY=xxx
GROK_API_KEY=xxx

# ============================================
# Market Data Providers
# ============================================

# Alpaca (US Stocks)
ALPACA_API_KEY=xxx
ALPACA_SECRET_KEY=xxx

# TwelveData (Forex, Metals)
TWELVEDATA_KEY=xxx

# ============================================
# Frontend Configuration
# ============================================

# Frontend development port
NOFX_FRONTEND_PORT=3000

# API base URL for frontend
VITE_API_URL=http://localhost:8080
```

### Configuration File

```go
// config/config.go
type Config struct {
    // Server
    APIServerPort int    `env:"API_SERVER_PORT" default:"8080"`

    // Database
    DBType     string `env:"DB_TYPE" default:"sqlite"`
    DBPath     string `env:"DB_PATH" default:"data/data.db"`
    DBHost     string `env:"DB_HOST"`
    DBPort     int    `env:"DB_PORT" default:"5432"`
    DBUser     string `env:"DB_USER"`
    DBPassword string `env:"DB_PASSWORD"`
    DBName     string `env:"DB_NAME"`

    // Security
    JWTSecret           string `env:"JWT_SECRET" required:"true"`
    DataEncryptionKey   string `env:"DATA_ENCRYPTION_KEY"`
    TransportEncryption bool   `env:"TRANSPORT_ENCRYPTION" default:"false"`
}
```

---

## Database Schema

### Core Tables

```sql
-- Traders: Trading bot configurations
CREATE TABLE traders (
    id              INTEGER PRIMARY KEY,
    name            TEXT NOT NULL,
    exchange        TEXT NOT NULL,
    api_key         TEXT,           -- Encrypted
    api_secret      TEXT,           -- Encrypted
    mode            TEXT DEFAULT 'paper',
    status          TEXT DEFAULT 'stopped',
    config          JSON,           -- Full strategy config
    created_at      DATETIME,
    updated_at      DATETIME
);

-- Positions: Open and historical positions
CREATE TABLE positions (
    id              INTEGER PRIMARY KEY,
    trader_id       INTEGER REFERENCES traders(id),
    symbol          TEXT NOT NULL,
    side            TEXT NOT NULL,  -- long/short
    size            REAL NOT NULL,
    entry_price     REAL NOT NULL,
    current_price   REAL,
    pnl             REAL,
    leverage        INTEGER,
    status          TEXT DEFAULT 'open',
    opened_at       DATETIME,
    closed_at       DATETIME
);

-- Decisions: AI decision history
CREATE TABLE decisions (
    id              INTEGER PRIMARY KEY,
    trader_id       INTEGER REFERENCES traders(id),
    timestamp       DATETIME NOT NULL,
    reasoning       TEXT,
    decisions       JSON,
    executed        BOOLEAN DEFAULT FALSE,
    created_at      DATETIME
);

-- Orders: Exchange order history
CREATE TABLE orders (
    id              INTEGER PRIMARY KEY,
    trader_id       INTEGER REFERENCES traders(id),
    decision_id     INTEGER REFERENCES decisions(id),
    symbol          TEXT NOT NULL,
    side            TEXT NOT NULL,
    type            TEXT NOT NULL,
    size            REAL NOT NULL,
    price           REAL,
    status          TEXT,
    exchange_id     TEXT,
    created_at      DATETIME
);

-- Users: Authentication
CREATE TABLE users (
    id              INTEGER PRIMARY KEY,
    username        TEXT UNIQUE NOT NULL,
    password_hash   TEXT NOT NULL,
    totp_secret     TEXT,           -- Encrypted
    is_admin        BOOLEAN DEFAULT FALSE,
    created_at      DATETIME
);

-- Backtests: Backtest runs
CREATE TABLE backtests (
    id              INTEGER PRIMARY KEY,
    name            TEXT,
    config          JSON,
    status          TEXT,
    metrics         JSON,
    started_at      DATETIME,
    completed_at    DATETIME
);

-- Debates: AI debate sessions
CREATE TABLE debates (
    id              INTEGER PRIMARY KEY,
    topic           TEXT,
    participants    JSON,
    rounds          JSON,
    result          JSON,
    created_at      DATETIME
);
```

### GORM Models

```go
// store/models.go
type Trader struct {
    gorm.Model
    Name       string
    Exchange   string
    APIKey     EncryptedString  // Auto-encrypted field
    APISecret  EncryptedString
    Mode       string
    Status     string
    Config     JSON
}

type Position struct {
    gorm.Model
    TraderID    uint
    Symbol      string
    Side        string
    Size        float64
    EntryPrice  float64
    PnL         float64
}
```

---

## Deployment Options

### Option 1: Local Development

```bash
# 1. Clone repository
git clone https://github.com/NoFxAiOS/nofx.git
cd nofx

# 2. Copy environment file
cp .env.example .env
# Edit .env with your configuration

# 3. Start backend
go run main.go

# 4. Start frontend (separate terminal)
cd web
npm install
npm run dev

# Access at http://localhost:3000
```

### Option 2: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  backend:
    build:
      context: .
      dockerfile: docker/Dockerfile.backend
    ports:
      - "8080:8080"
    volumes:
      - ./data:/app/data
    env_file:
      - .env

  frontend:
    build:
      context: ./web
      dockerfile: ../docker/Dockerfile.frontend
    ports:
      - "3000:3000"
    environment:
      - VITE_API_URL=http://backend:8080
    depends_on:
      - backend
```

```bash
# Start all services
docker compose up -d

# View logs
docker compose logs -f

# Stop services
docker compose down
```

### Option 3: Production Docker

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  nofx:
    image: ghcr.io/nofxaios/nofx:latest
    ports:
      - "8080:8080"
    volumes:
      - nofx-data:/app/data
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - DATA_ENCRYPTION_KEY=${DATA_ENCRYPTION_KEY}
      - DB_TYPE=postgres
      - DB_HOST=db
      - DB_PORT=5432
      - DB_USER=nofx
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=nofx
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=nofx
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=nofx

volumes:
  nofx-data:
  postgres-data:
```

### Option 4: Railway (One-Click)

NOFX supports Railway deployment:

1. Fork the repository
2. Connect to Railway
3. Set environment variables
4. Deploy

Railway automatically detects the Dockerfile and builds.

---

## Build Process

### Backend Build

```bash
# Development
go run main.go

# Production build
go build -o nofx -ldflags="-s -w" main.go

# Cross-compilation
GOOS=linux GOARCH=amd64 go build -o nofx-linux main.go
GOOS=darwin GOARCH=arm64 go build -o nofx-darwin main.go
GOOS=windows GOARCH=amd64 go build -o nofx.exe main.go
```

### Frontend Build

```bash
cd web

# Development
npm run dev

# Production build
npm run build

# Preview production build
npm run preview
```

### Docker Build

```dockerfile
# docker/Dockerfile.backend
FROM golang:1.25-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o nofx main.go

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/nofx .
COPY --from=builder /app/.env.example .env
EXPOSE 8080
CMD ["./nofx"]
```

---

## Monitoring & Logging

### Logging Configuration

```go
// logger/logger.go
func Init() {
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnix

    // Console output in development
    if os.Getenv("ENV") == "development" {
        log.Logger = log.Output(zerolog.ConsoleWriter{Out: os.Stderr})
    }

    // JSON output in production
    // Logs go to stdout for container collection
}
```

### Log Levels

| Level | Usage |
|-------|-------|
| DEBUG | Detailed debugging info |
| INFO | General operational info |
| WARN | Potential issues |
| ERROR | Errors that don't stop operation |
| FATAL | Critical errors, shutdown |

### Health Check

```go
// api/server.go
func healthCheck(c *gin.Context) {
    c.JSON(200, gin.H{
        "status": "ok",
        "version": version,
        "uptime": time.Since(startTime).String(),
    })
}

// GET /api/health
```

---

## Performance Considerations

### Database

- SQLite for single-instance deployments
- PostgreSQL for high-availability setups
- Connection pooling enabled by default
- Indexes on frequently queried columns

### API

- Request rate limiting (100 req/min default)
- Response compression (gzip)
- Connection keep-alive
- Graceful shutdown handling

### Frontend

- Code splitting via Vite
- Lazy loading for routes
- SWR caching for API data
- Optimistic UI updates

---

## Next Steps

- [System Overview](01-system-overview.md) - High-level architecture
- [Trading Core Flow](02-trading-core.md) - How trading decisions are made
- [Security Architecture](04-security.md) - Encryption and authentication

---

[← Back to Architecture](README.md)

# System Overview

**Language:** English | [中文](01-system-overview.zh-CN.md)

Complete system architecture overview for NOFX platform.

---

## High-Level Architecture

NOFX is a full-stack AI-powered trading platform with the following major subsystems:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              NOFX Platform                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌───────────────────────────────────────────────────────────────────────┐     │
│   │                         Presentation Layer                             │     │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │     │
│   │  │  Dashboard  │  │  Strategy   │  │  Backtest   │  │   Debate    │   │     │
│   │  │   (Home)    │  │   Studio    │  │   Center    │  │    Arena    │   │     │
│   │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │     │
│   │                      React + TypeScript + Vite                         │     │
│   └───────────────────────────────────────────────────────────────────────┘     │
│                                      │                                           │
│                                      │ REST API + SSE + WebSocket                │
│                                      ▼                                           │
│   ┌───────────────────────────────────────────────────────────────────────┐     │
│   │                           API Layer (Gin)                              │     │
│   │  /api/traders  /api/backtest  /api/debate  /api/models  /api/auth     │     │
│   └───────────────────────────────────────────────────────────────────────┘     │
│                                      │                                           │
│         ┌────────────────────────────┼────────────────────────────┐             │
│         │                            │                            │             │
│         ▼                            ▼                            ▼             │
│   ┌───────────┐              ┌───────────────┐             ┌───────────┐        │
│   │  Trader   │              │    Backtest   │             │  Debate   │        │
│   │  Manager  │              │    Manager    │             │  Engine   │        │
│   └─────┬─────┘              └───────┬───────┘             └─────┬─────┘        │
│         │                            │                            │             │
│         └────────────────────────────┼────────────────────────────┘             │
│                                      │                                           │
│                                      ▼                                           │
│   ┌───────────────────────────────────────────────────────────────────────┐     │
│   │                         Core Services                                  │     │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │     │
│   │  │   Kernel    │  │   Market    │  │     MCP     │  │   Crypto    │   │     │
│   │  │   Engine    │  │    Data     │  │   Clients   │  │  Service    │   │     │
│   │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │     │
│   └───────────────────────────────────────────────────────────────────────┘     │
│                                      │                                           │
│         ┌────────────────────────────┼────────────────────────────┐             │
│         │                            │                            │             │
│         ▼                            ▼                            ▼             │
│   ┌───────────┐              ┌───────────────┐             ┌───────────┐        │
│   │ Exchange  │              │   Database    │             │    AI     │        │
│   │  Clients  │              │    (GORM)     │             │ Providers │        │
│   └───────────┘              └───────────────┘             └───────────┘        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Directory Structure

```
nofx/
├── main.go                    # Application entry point
├── go.mod / go.sum            # Go dependencies
├── Makefile                   # Build commands
│
├── api/                       # HTTP API layer
│   └── server.go              # All REST endpoints (121KB)
│
├── trader/                    # Live trading execution
│   ├── auto_trader.go         # Main trader logic (77KB)
│   ├── executor_*.go          # Exchange-specific executors
│   └── types.go               # Shared types
│
├── kernel/                    # Trading decision engine
│   ├── engine.go              # Core engine (68KB)
│   ├── prompt_builder.go      # AI prompt construction
│   └── data_builder.go        # Market data assembly
│
├── backtest/                  # Historical simulation
│   ├── manager.go             # Backtest orchestration
│   ├── runner.go              # Simulation execution
│   └── metrics.go             # Performance calculations
│
├── debate/                    # Multi-AI consensus
│   └── engine.go              # Debate logic (46KB)
│
├── market/                    # Market data service
│   └── data.go                # K-lines, indicators
│
├── mcp/                       # AI model integrations
│   ├── client.go              # Base client
│   ├── deepseek.go            # DeepSeek provider
│   ├── qwen.go                # Qwen provider
│   └── ...                    # Other AI providers
│
├── provider/                  # External data providers
│   ├── coinank.go             # CoinAnk API
│   ├── alpaca.go              # Alpaca (stocks)
│   └── twelvedata.go          # TwelveData (forex)
│
├── store/                     # Database layer
│   ├── store.go               # Repository interfaces
│   └── gorm.go                # GORM implementation
│
├── auth/                      # Authentication
│   └── auth.go                # JWT + 2FA
│
├── crypto/                    # Encryption service
│   └── crypto.go              # AES-256 + RSA-4096
│
├── config/                    # Configuration
│   └── config.go              # Environment loading
│
├── logger/                    # Logging
│   └── logger.go              # Zerolog setup
│
├── manager/                   # Multi-trader management
│   └── trader_manager.go      # Trader lifecycle
│
└── web/                       # React frontend
    ├── src/
    │   ├── pages/             # Page components
    │   ├── components/        # Shared UI components
    │   ├── lib/api.ts         # API client
    │   └── stores/            # Zustand state
    ├── package.json           # Node dependencies
    └── vite.config.ts         # Build configuration
```

---

## Module Relationships

```
                                    ┌─────────────┐
                                    │   main.go   │
                                    └──────┬──────┘
                                           │
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
              ▼                            ▼                            ▼
       ┌────────────┐              ┌───────────────┐             ┌───────────┐
       │   config   │              │     store     │             │   logger  │
       │            │              │               │             │           │
       │ - Load env │              │ - GORM init   │             │ - Zerolog │
       │ - Validate │              │ - Migrations  │             │ - Levels  │
       └────────────┘              └───────┬───────┘             └───────────┘
                                           │
                                           ▼
       ┌────────────┐              ┌───────────────┐             ┌───────────┐
       │   crypto   │◄─────────────│    manager    │─────────────►│    api    │
       │            │              │               │             │           │
       │ - AES-256  │              │ - CreateTrader│             │ - Routes  │
       │ - RSA-4096 │              │ - LoadTraders │             │ - Handlers│
       └────────────┘              │ - StopTrader  │             │ - Auth    │
                                   └───────┬───────┘             └───────────┘
                                           │
                                           ▼
                                   ┌───────────────┐
                                   │    trader     │
                                   │               │
                                   │ - AutoTrader  │
                                   │ - Executors   │
                                   └───────┬───────┘
                                           │
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
              ▼                            ▼                            ▼
       ┌────────────┐              ┌───────────────┐             ┌───────────┐
       │   kernel   │              │    market     │             │    mcp    │
       │            │              │               │             │           │
       │ - Engine   │◄─────────────│ - K-lines     │             │ - AI API  │
       │ - Prompts  │              │ - Indicators  │             │ - Models  │
       │ - Parsing  │              │ - CoinAnk     │             │ - Request │
       └────────────┘              └───────────────┘             └───────────┘
```

---

## Component Descriptions

### Presentation Layer (Frontend)

| Component | Purpose | Key Features |
|-----------|---------|--------------|
| **Dashboard** | Trading overview | Active traders, P&L summary, recent decisions |
| **Strategy Studio** | Strategy configuration | Coin selection, risk settings, AI model config |
| **Backtest Center** | Historical testing | Multi-symbol backtesting, performance metrics |
| **Debate Arena** | Multi-AI consensus | 5 AI personalities, voting, live streaming |

### API Layer

Single file (`api/server.go`) containing all REST endpoints:

| Endpoint Group | Purpose |
|----------------|---------|
| `/api/auth/*` | Authentication (login, register, 2FA) |
| `/api/traders/*` | Trader CRUD, start/stop, status |
| `/api/backtest/*` | Run backtests, get results |
| `/api/debate/*` | Create/run debates |
| `/api/models/*` | AI model management |
| `/api/exchanges/*` | Exchange connections |
| `/api/strategies/*` | Strategy templates |

### Core Services

| Service | Responsibility | Key Files |
|---------|----------------|-----------|
| **Kernel Engine** | Build prompts, parse AI decisions, apply risk rules | `kernel/engine.go` |
| **Market Data** | Fetch K-lines, calculate indicators | `market/data.go` |
| **MCP Clients** | Communicate with AI providers | `mcp/*.go` |
| **Crypto Service** | Encrypt/decrypt sensitive data | `crypto/crypto.go` |

### Data Layer

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Database** | SQLite (default) / PostgreSQL | Persistent storage |
| **ORM** | GORM | Object-relational mapping |
| **Models** | Go structs | Traders, Positions, Decisions, Orders |

---

## Supported Integrations

### Exchanges (CEX)

| Exchange | Trading | Data | Implementation |
|----------|---------|------|----------------|
| Binance Futures | ✅ | ✅ | `go-binance/v2` |
| Bybit | ✅ | ✅ | `bybit.go.api` |
| OKX | ✅ | ✅ | Custom SDK |
| Bitget | ✅ | ✅ | Custom SDK |

### Exchanges (DEX)

| Exchange | Trading | Data | Implementation |
|----------|---------|------|----------------|
| Hyperliquid | ✅ | ✅ | Custom SDK |
| Aster | ✅ | ✅ | Custom SDK |
| Lighter | ✅ | ✅ | Custom SDK |

### Traditional Markets

| Provider | Assets | Implementation |
|----------|--------|----------------|
| Alpaca | US Stocks | Custom client |
| TwelveData | Forex, Metals | Custom client |

### AI Models

| Provider | Models | Implementation |
|----------|--------|----------------|
| DeepSeek | DeepSeek-V3, DeepSeek-R1 | `mcp/deepseek.go` |
| Alibaba | Qwen-Max, Qwen-Plus | `mcp/qwen.go` |
| OpenAI | GPT-4o, GPT-o1 | `mcp/openai.go` |
| Anthropic | Claude Sonnet/Opus | `mcp/claude.go` |
| Google | Gemini Pro/Flash | `mcp/gemini.go` |
| xAI | Grok | `mcp/grok.go` |
| Moonshot | Kimi | `mcp/kimi.go` |

### Data Providers

| Provider | Data Type | Usage |
|----------|-----------|-------|
| CoinAnk | OI rankings, funding rates, depth | Primary crypto data |
| Exchange APIs | K-lines, account, positions | Direct exchange data |

---

## Communication Patterns

### REST API

Standard request-response for CRUD operations:

```
Frontend → POST /api/traders → Backend → Database
Frontend ← JSON Response ← Backend
```

### Server-Sent Events (SSE)

Real-time streaming for long-running operations:

```
Frontend → GET /api/backtest/{id}/stream → Backend
Frontend ← SSE: progress, metrics ← Backend (continuous)
Frontend ← SSE: complete ← Backend (final)
```

### WebSocket

Real-time market data and position updates:

```
Frontend ↔ WS /ws/market ↔ Backend ↔ CoinAnk WebSocket
```

---

## Next Steps

- [Trading Core Flow](02-trading-core.md) - How trading decisions are made
- [Tech Stack & Deployment](03-tech-stack.md) - Technologies and deployment
- [Security Architecture](04-security.md) - Encryption and authentication

---

[← Back to Architecture](README.md)

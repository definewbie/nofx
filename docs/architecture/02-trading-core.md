# Trading Core Flow

**Language:** English | [中文](02-trading-core.zh-CN.md)

Deep dive into how NOFX makes and executes trading decisions.

---

## Trading Cycle Overview

Every trading cycle follows this sequence:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Trading Cycle (Every N Minutes)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                 │
│  │ 1. Select    │     │ 2. Fetch     │     │ 3. Build     │                 │
│  │    Coins     │────►│    Market    │────►│    Trading   │                 │
│  │              │     │    Data      │     │    Context   │                 │
│  └──────────────┘     └──────────────┘     └──────────────┘                 │
│                                                   │                          │
│                                                   ▼                          │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                 │
│  │ 6. Execute   │     │ 5. Parse     │     │ 4. Call      │                 │
│  │    Orders    │◄────│    Response  │◄────│    AI Model  │                 │
│  │              │     │              │     │              │                 │
│  └──────────────┘     └──────────────┘     └──────────────┘                 │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────┐                                                           │
│  │ 7. Record    │                                                           │
│  │    Decision  │                                                           │
│  └──────────────┘                                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Coin Selection

The trader selects which coins to analyze based on the configured coin source.

### Coin Sources

| Source | Description | Use Case |
|--------|-------------|----------|
| **Static** | Fixed list of symbols | Conservative, known assets |
| **AI500** | CoinAnk AI-ranked top 500 | Broad market coverage |
| **OI High** | Highest open interest | Long opportunities |
| **OI Low** | Lowest open interest | Short opportunities |
| **Mixed** | Combines multiple sources | Balanced approach |

### Selection Flow

```go
// trader/auto_trader.go
func (t *AutoTrader) selectCoins() []string {
    switch t.config.CoinSource {
    case "static":
        return t.config.StaticCoins
    case "ai500":
        return t.provider.GetAI500Pool()
    case "oi_high":
        return t.provider.GetOIRanking("high", t.config.CoinCount)
    case "oi_low":
        return t.provider.GetOIRanking("low", t.config.CoinCount)
    case "mixed":
        return t.getMixedCoins()
    }
}
```

---

## Step 2: Market Data Fetch

For each selected coin, fetch comprehensive market data.

### Data Components

| Data Type | Source | Timeframes |
|-----------|--------|------------|
| K-lines (OHLCV) | Exchange API | 5m, 15m, 1h, 4h |
| Technical Indicators | Calculated | EMA, MACD, RSI, ATR, Bollinger |
| Open Interest | CoinAnk API | Current + historical |
| Funding Rate | CoinAnk API | Current rate |
| Order Book Depth | CoinAnk WebSocket | Real-time |

### Data Assembly

```go
// kernel/data_builder.go
type TradingContext struct {
    Account     AccountInfo      // Balance, equity, margin
    Positions   []Position       // Current open positions
    Coins       []CoinData       // Market data per coin
    History     []RecentTrade    // Recent trade history
    Timestamp   time.Time
}

type CoinData struct {
    Symbol      string
    Klines      map[string][]Kline  // Timeframe → candles
    Indicators  Indicators          // EMA, MACD, RSI, ATR
    OI          OpenInterestData    // OI value + change
    FundingRate float64
    Depth       DepthData           // Bid/ask spread
}
```

---

## Step 3: Prompt Construction

The Kernel Engine builds structured prompts for AI decision-making.

### Prompt Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                        SYSTEM PROMPT                             │
├─────────────────────────────────────────────────────────────────┤
│ Role: You are an AI trading assistant                           │
│ Trading Mode: {scalping | swing | position}                     │
│ Risk Rules:                                                     │
│   - Max position size: {value}                                  │
│   - Max leverage: {value}                                       │
│   - Max drawdown: {value}                                       │
│ Output Format: XML <reasoning> + JSON <decision>                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         USER PROMPT                              │
├─────────────────────────────────────────────────────────────────┤
│ ## Account Status                                               │
│ Balance: $10,000 | Equity: $10,500 | Margin: 15%                │
│                                                                  │
│ ## Current Positions                                            │
│ | Symbol | Side | Size | Entry | PnL |                          │
│ | BTCUSDT | LONG | 0.1 | 42000 | +$150 |                        │
│                                                                  │
│ ## Coin Analysis                                                │
│ ### ETHUSDT                                                     │
│ Price: $2,500                                                   │
│ K-lines (5m): [OHLCV data...]                                   │
│ Indicators: EMA(20)=2480, RSI=65, MACD=bullish                  │
│ OI: $5.2B (+3.5%)                                               │
│ Funding: 0.01%                                                  │
│                                                                  │
│ ## Recent Trades (last 24h)                                     │
│ [Trade history...]                                              │
│                                                                  │
│ ## Instructions                                                 │
│ Analyze the market and provide trading decisions.               │
└─────────────────────────────────────────────────────────────────┘
```

### Section Configuration

Users can enable/disable prompt sections:

| Section | Content | Configurable |
|---------|---------|--------------|
| Account Status | Balance, equity, margin | ✅ |
| Positions | Current open positions | ✅ |
| K-line Data | OHLCV candles | ✅ per timeframe |
| Indicators | EMA, MACD, RSI, ATR | ✅ per indicator |
| OI Data | Open interest metrics | ✅ |
| Funding Rate | Funding rate info | ✅ |
| Trade History | Recent trades | ✅ |

---

## Step 4: AI Model Call

The MCP client sends the prompt to the configured AI model.

### Request Flow

```go
// mcp/client.go
type AIRequest struct {
    Model       string
    Messages    []Message
    Temperature float64
    MaxTokens   int
}

func (c *Client) Call(ctx context.Context, req AIRequest) (*AIResponse, error) {
    // 1. Select provider based on model
    provider := c.getProvider(req.Model)

    // 2. Build provider-specific request
    httpReq := provider.BuildRequest(req)

    // 3. Send with retry (3 attempts, 120s timeout)
    resp, err := c.sendWithRetry(ctx, httpReq)

    // 4. Parse provider-specific response
    return provider.ParseResponse(resp)
}
```

### Supported Models

```go
// Model routing
switch {
case strings.HasPrefix(model, "deepseek"):
    return DeepSeekProvider
case strings.HasPrefix(model, "qwen"):
    return QwenProvider
case strings.HasPrefix(model, "gpt"):
    return OpenAIProvider
case strings.HasPrefix(model, "claude"):
    return ClaudeProvider
// ... more providers
}
```

---

## Step 5: Response Parsing

Parse the AI response to extract reasoning and trading decisions.

### Expected Response Format

```xml
<reasoning>
Market analysis shows BTC is in an uptrend with strong momentum.
RSI at 65 indicates bullish strength without overbought conditions.
OI increasing suggests new money entering the market.
Recommend opening a long position with tight stop loss.
</reasoning>

<decision>
[
  {
    "symbol": "BTCUSDT",
    "action": "open",
    "side": "long",
    "size": 0.05,
    "leverage": 10,
    "reason": "Uptrend continuation with strong OI support"
  }
]
</decision>
```

### Parsing Logic

```go
// kernel/engine.go
func parseFullDecisionResponse(response string) (*DecisionResult, error) {
    // 1. Extract reasoning from <reasoning> tags
    reasoning := extractTag(response, "reasoning")

    // 2. Extract decision JSON from <decision> tags
    decisionJSON := extractTag(response, "decision")

    // 3. Parse JSON into decision structs
    var decisions []Decision
    json.Unmarshal([]byte(decisionJSON), &decisions)

    // 4. Validate decision structure
    for _, d := range decisions {
        if err := validateDecision(d); err != nil {
            return nil, err
        }
    }

    return &DecisionResult{
        Reasoning: reasoning,
        Decisions: decisions,
    }, nil
}
```

### Decision Structure

```go
type Decision struct {
    Symbol   string  `json:"symbol"`
    Action   string  `json:"action"`   // open, close, hold, adjust
    Side     string  `json:"side"`     // long, short
    Size     float64 `json:"size"`     // Position size
    Leverage int     `json:"leverage"` // Leverage multiplier
    Reason   string  `json:"reason"`   // AI's reasoning
}
```

---

## Step 6: Order Execution

Apply risk controls and execute orders on the exchange.

### Execution Pipeline

```
┌──────────────────────────────────────────────────────────────────┐
│                     Execution Pipeline                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────┐                                                 │
│  │ AI Decision │                                                 │
│  └──────┬──────┘                                                 │
│         │                                                         │
│         ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Risk Control Layer                        │ │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐ │ │
│  │  │ Position  │  │ Leverage  │  │ Drawdown  │  │ Exposure  │ │ │
│  │  │   Limit   │  │   Check   │  │   Check   │  │   Check   │ │ │
│  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘ │ │
│  └──────────────────────────┬──────────────────────────────────┘ │
│                             │                                     │
│                             ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Order Sorting                             │ │
│  │  1. Close positions first (reduce risk)                      │ │
│  │  2. Adjust positions second                                  │ │
│  │  3. Open new positions last                                  │ │
│  └──────────────────────────┬──────────────────────────────────┘ │
│                             │                                     │
│                             ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                  Exchange Executor                           │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │ │
│  │  │ Binance │  │  Bybit  │  │   OKX   │  │Hyperliquid│      │ │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │ │
│  └──────────────────────────┬──────────────────────────────────┘ │
│                             │                                     │
│                             ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Order Submission                          │ │
│  │  - Market order (default)                                    │ │
│  │  - Limit order (optional)                                    │ │
│  │  - Stop loss / Take profit                                   │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Risk Controls

| Control | Rule | Applied |
|---------|------|---------|
| Max Position Size | Size ≤ config.maxPositionSize | Per symbol |
| Max Leverage | Leverage ≤ config.maxLeverage | Per order |
| Max Total Exposure | Sum(positions) ≤ config.maxExposure | Portfolio |
| Max Drawdown | Equity loss ≤ config.maxDrawdown | Stop trading |
| Min Balance | Available ≥ config.minBalance | Skip order |

### Exchange Executor Interface

```go
// trader/executor.go
type Executor interface {
    // Account
    GetBalance(ctx context.Context) (*Balance, error)
    GetPositions(ctx context.Context) ([]Position, error)

    // Orders
    PlaceOrder(ctx context.Context, order Order) (*OrderResult, error)
    CancelOrder(ctx context.Context, orderId string) error

    // Market data
    GetKlines(ctx context.Context, symbol string, interval string) ([]Kline, error)
}
```

---

## Step 7: Decision Recording

Store the decision and execution results in the database.

### Stored Data

```go
// store/models.go
type DecisionRecord struct {
    ID          uint
    TraderID    uint
    Timestamp   time.Time

    // AI Decision
    Reasoning   string
    Decisions   JSON         // Full decision array

    // Execution
    Executed    bool
    Orders      JSON         // Order results
    Error       string       // Error if failed

    // Context
    AccountInfo JSON         // Balance at decision time
    Positions   JSON         // Positions at decision time
    MarketData  JSON         // Summary of market data
}
```

### Decision Lifecycle

```
Created → Parsed → Validated → Executed → Recorded
   │         │         │           │          │
   │         │         │           │          └── Stored in DB
   │         │         │           └── Orders submitted
   │         │         └── Risk checks passed
   │         └── Valid JSON structure
   └── AI response received
```

---

## Execution Modes

### Live Trading

Real orders executed on exchanges.

```go
trader.Mode = "live"
// Orders are submitted to exchange
// Real money at risk
```

### Paper Trading

Simulated execution without real orders.

```go
trader.Mode = "paper"
// Orders are simulated
// No real money involved
// Useful for testing strategies
```

### Backtest Mode

Historical simulation using past data.

```go
// backtest/runner.go
// Replay historical K-lines
// Simulate AI decisions
// Calculate metrics
```

---

## Trading Modes

AI prompts are adjusted based on trading mode:

| Mode | Holding Period | Stop Loss | Take Profit |
|------|----------------|-----------|-------------|
| **Scalping** | Minutes to hours | Tight (1-2%) | Quick (2-5%) |
| **Swing** | Hours to days | Medium (3-5%) | Medium (5-10%) |
| **Position** | Days to weeks | Wide (5-10%) | Large (10-20%) |

---

## Error Handling

### Retry Strategy

| Error Type | Action | Max Retries |
|------------|--------|-------------|
| Network timeout | Retry with backoff | 3 |
| Rate limit | Wait and retry | 3 |
| Invalid response | Log and skip | 1 |
| Execution failure | Record error, continue | 1 |

### Failure Recovery

```go
// trader/auto_trader.go
func (t *AutoTrader) runCycle() {
    defer func() {
        if r := recover(); r != nil {
            t.logger.Error("Cycle panic recovered", "error", r)
            // Continue to next cycle
        }
    }()

    // Trading cycle logic...
}
```

---

## Next Steps

- [System Overview](01-system-overview.md) - High-level architecture
- [Tech Stack & Deployment](03-tech-stack.md) - Technologies and deployment
- [Security Architecture](04-security.md) - Encryption and authentication

---

[← Back to Architecture](README.md)

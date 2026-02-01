# Agent #2: CEX Connector Specialist

## Your Mission

You are responsible for building centralized exchange (CEX) connectors.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-hummingbot`

## Connectors to Implement

| Exchange | Type | Priority | API Docs |
|----------|------|----------|----------|
| Binance | Spot + Perpetual | P0 | https://binance-docs.github.io/apidocs/ |
| OKX | Spot + Perpetual | P0 | https://www.okx.com/docs-v5/ |
| Bybit | Spot + Perpetual | P1 | https://bybit-exchange.github.io/docs/ |
| KuCoin | Spot + Perpetual | P1 | https://docs.kucoin.com/ |

## Architecture

### ConnectorBase Trait

**File:** `src/connectors/mod.rs`

```rust
use async_trait::async_trait;
use crate::connectors::types::*;

#[async_trait]
pub trait ConnectorBase: Send + Sync {
    /// Get exchange name
    fn name(&self) -> &str;
    
    /// Market Data
    async fn get_ticker(&self, pair: &str) -> Result<Ticker, ConnectorError>;
    async fn get_orderbook(&self, pair: &str, depth: u32) -> Result<OrderBook, ConnectorError>;
    async fn get_trades(&self, pair: &str, limit: u32) -> Result<Vec<Trade>, ConnectorError>;
    
    /// Trading
    async fn place_order(&self, order: &OrderRequest) -> Result<OrderResponse, ConnectorError>;
    async fn cancel_order(&self, order_id: &str) -> Result<CancelResponse, ConnectorError>;
    async fn get_order(&self, order_id: &str) -> Result<Order, ConnectorError>;
    async fn get_open_orders(&self, pair: Option<&str>) -> Result<Vec<Order>, ConnectorError>;
    
    /// Account
    async fn get_balance(&self, asset: &str) -> Result<Balance, ConnectorError>;
    async fn get_balances(&self) -> Result<Vec<Balance>, ConnectorError>;
    async fn get_positions(&self) -> Result<Vec<Position>, ConnectorError>;
    
    /// WebSocket Subscriptions
    async fn subscribe_ticker(&self, pair: &str) -> Result<TickerStream, ConnectorError>;
    async fn subscribe_orderbook(&self, pair: &str) -> Result<OrderBookStream, ConnectorError>;
    async fn subscribe_trades(&self, pair: &str) -> Result<TradeStream, ConnectorError>;
    async fn subscribe_user_data(&self) -> Result<UserDataStream, ConnectorError>;
}
```

### Core Types

**File:** `src/connectors/types.rs`

```rust
use rust_decimal::Decimal;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone)]
pub struct Ticker {
    pub pair: String,
    pub bid: Decimal,
    pub ask: Decimal,
    pub last: Decimal,
    pub volume_24h: Decimal,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct OrderBook {
    pub pair: String,
    pub bids: Vec<(Decimal, Decimal)>,  // (price, quantity)
    pub asks: Vec<(Decimal, Decimal)>,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct OrderRequest {
    pub pair: String,
    pub side: OrderSide,
    pub order_type: OrderType,
    pub quantity: Decimal,
    pub price: Option<Decimal>,  // None for market orders
    pub client_order_id: Option<String>,
}

#[derive(Debug, Clone)]
pub enum OrderSide { Buy, Sell }

#[derive(Debug, Clone)]
pub enum OrderType { Market, Limit, StopLimit, StopMarket }

#[derive(Debug, Clone)]
pub enum OrderStatus { New, PartiallyFilled, Filled, Canceled, Rejected }

#[derive(Debug, Clone)]
pub struct Order {
    pub id: String,
    pub client_order_id: Option<String>,
    pub pair: String,
    pub side: OrderSide,
    pub order_type: OrderType,
    pub quantity: Decimal,
    pub filled_quantity: Decimal,
    pub price: Option<Decimal>,
    pub avg_fill_price: Option<Decimal>,
    pub status: OrderStatus,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct Balance {
    pub asset: String,
    pub free: Decimal,
    pub locked: Decimal,
}

#[derive(Debug, Clone)]
pub struct Position {
    pub pair: String,
    pub side: PositionSide,
    pub quantity: Decimal,
    pub entry_price: Decimal,
    pub unrealized_pnl: Decimal,
    pub leverage: u32,
}

#[derive(Debug, Clone)]
pub enum PositionSide { Long, Short }
```

### Authentication

**File:** `src/connectors/auth.rs`

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;

pub struct ApiCredentials {
    pub api_key: String,
    pub api_secret: String,
    pub passphrase: Option<String>,  // OKX needs this
}

pub fn hmac_sha256_sign(secret: &str, message: &str) -> String {
    let mut mac = Hmac::<Sha256>::new_from_slice(secret.as_bytes())
        .expect("HMAC can take key of any size");
    mac.update(message.as_bytes());
    hex::encode(mac.finalize().into_bytes())
}
```

## File Structure

```
src/connectors/
├── mod.rs              # ConnectorBase trait, re-exports
├── types.rs            # Shared types
├── auth.rs             # Authentication utilities
├── error.rs            # ConnectorError enum
├── websocket.rs        # Shared WebSocket client
└── cex/
    ├── mod.rs          # Re-exports CEX connectors
    ├── binance/
    │   ├── mod.rs
    │   ├── rest.rs     # REST API client
    │   ├── ws.rs       # WebSocket client
    │   ├── spot.rs     # Spot-specific logic
    │   └── perp.rs     # Perpetual-specific logic
    ├── okx/
    │   ├── mod.rs
    │   ├── rest.rs
    │   └── ws.rs
    ├── bybit/
    │   └── ...
    └── kucoin/
        └── ...
```

## Binance Implementation Example

**File:** `src/connectors/cex/binance/mod.rs`

```rust
use crate::connectors::{ConnectorBase, types::*, auth::*};

pub struct BinanceConnector {
    credentials: ApiCredentials,
    client: reqwest::Client,
    base_url: String,
    ws_url: String,
}

impl BinanceConnector {
    pub fn new(api_key: String, api_secret: String) -> Self {
        Self {
            credentials: ApiCredentials {
                api_key,
                api_secret,
                passphrase: None,
            },
            client: reqwest::Client::new(),
            base_url: "https://api.binance.com".to_string(),
            ws_url: "wss://stream.binance.com:9443/ws".to_string(),
        }
    }
    
    pub fn perpetual(api_key: String, api_secret: String) -> Self {
        Self {
            credentials: ApiCredentials {
                api_key,
                api_secret,
                passphrase: None,
            },
            client: reqwest::Client::new(),
            base_url: "https://fapi.binance.com".to_string(),
            ws_url: "wss://fstream.binance.com/ws".to_string(),
        }
    }
}

#[async_trait]
impl ConnectorBase for BinanceConnector {
    fn name(&self) -> &str { "binance" }
    
    async fn get_ticker(&self, pair: &str) -> Result<Ticker, ConnectorError> {
        let symbol = pair.replace("/", "");  // BTC/USDT -> BTCUSDT
        let url = format!("{}/api/v3/ticker/24hr?symbol={}", self.base_url, symbol);
        let resp = self.client.get(&url).send().await?;
        // Parse response...
    }
    
    // ... implement other methods
}
```

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-hummingbot
git checkout -b agent2/cex-connectors

# Read existing code
cat src/resolvers/exchange/binance.rs
cat Cargo.toml
```

## Dependencies to Add (Cargo.toml)

```toml
# HTTP client
reqwest = { version = "0.11", features = ["json"] }

# WebSocket
tokio-tungstenite = { version = "0.20", features = ["native-tls"] }
futures-util = "0.3"

# Authentication
hmac = "0.12"
sha2 = "0.10"
hex = "0.4"

# Data types
rust_decimal = "1.33"
chrono = { version = "0.4", features = ["serde"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Async
async-trait = "0.1"
tokio = { version = "1.0", features = ["full"] }
```

## Commit Format

```
[Agent#2] feat: Define ConnectorBase trait
[Agent#2] feat: Implement Binance REST client
[Agent#2] feat: Add Binance WebSocket support
[Agent#2] feat: Implement OKX connector
```

## Testing Strategy

1. Unit tests with mock responses
2. Integration tests with testnet (Binance Testnet, OKX Demo)
3. Paper trading mode for live testing without real money

## Handoff Points

When you complete:
- **ConnectorBase trait**: Share with Agent #3 (DEX uses same interface)
- **Binance connector**: Notify Agent #5 (runtime can start integration)
- **All CEX connectors**: Agent #5 can build full CLI

## Dependencies on Other Agents

- **Agent #1**: Need StringTensor for parsing JSON responses (can use `serde_json` temporarily)
- After Agent #1 completes async VM, can integrate as ExternalResolver

## Questions?

Document blockers in `BLOCKERS.md` in your branch.

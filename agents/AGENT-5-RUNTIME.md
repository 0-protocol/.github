# Agent #5: Runtime & Infrastructure Developer

## Your Mission

You are responsible for building the trading runtime, backtesting engine, paper trading, and CLI.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-hummingbot`

## Components to Build

| Component | Priority | Description |
|-----------|----------|-------------|
| TradingRuntime | P0 | Tick-based strategy execution |
| Order Tracker | P0 | Order lifecycle management |
| Paper Trading | P1 | Simulated trading mode |
| Backtesting | P1 | Historical data replay |
| CLI | P1 | Full command interface |
| Logging | P2 | Structured logging |

## File Structure

```
src/
├── runtime/
│   ├── mod.rs           # TradingRuntime struct
│   ├── tick.rs          # Tick-based execution loop
│   ├── events.rs        # Event handling
│   └── state.rs         # Runtime state management
├── backtest/
│   ├── mod.rs           # Backtesting engine
│   ├── data.rs          # Historical data loading (CSV, Parquet)
│   ├── simulator.rs     # Exchange simulator
│   └── metrics.rs       # Performance metrics
├── paper/
│   ├── mod.rs           # Paper trading mode
│   └── mock_exchange.rs # Mock exchange with slippage
├── orders/
│   ├── mod.rs           # Order management
│   ├── tracker.rs       # Order tracking
│   └── lifecycle.rs     # Order state machine
├── logging/
│   └── mod.rs           # Structured logging with tracing
└── main.rs              # CLI with clap
```

## TradingRuntime

**File:** `src/runtime/mod.rs`

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;
use zerolang::{VM, RuntimeGraph};
use crate::connectors::ConnectorBase;
use crate::orders::OrderTracker;

pub struct TradingRuntime {
    /// The 0-lang VM
    vm: VM,
    
    /// Connected exchanges
    connectors: HashMap<String, Arc<dyn ConnectorBase>>,
    
    /// Active strategies
    active_strategies: Vec<ActiveStrategy>,
    
    /// Order tracking
    order_tracker: Arc<RwLock<OrderTracker>>,
    
    /// Runtime state
    state: RuntimeState,
    
    /// Tick interval (default: 1 second)
    tick_interval: std::time::Duration,
}

pub struct ActiveStrategy {
    pub id: String,
    pub graph: RuntimeGraph,
    pub connector: String,
    pub pair: String,
    pub status: StrategyStatus,
}

pub enum StrategyStatus {
    Running,
    Paused,
    Stopped,
    Error(String),
}

impl TradingRuntime {
    pub fn new() -> Self {
        Self {
            vm: VM::new(),
            connectors: HashMap::new(),
            active_strategies: Vec::new(),
            order_tracker: Arc::new(RwLock::new(OrderTracker::new())),
            state: RuntimeState::default(),
            tick_interval: std::time::Duration::from_secs(1),
        }
    }
    
    /// Add a connector
    pub fn add_connector(&mut self, name: &str, connector: Arc<dyn ConnectorBase>) {
        self.connectors.insert(name.to_string(), connector);
    }
    
    /// Load a strategy from file
    pub fn load_strategy(&mut self, path: &std::path::Path) -> Result<String, RuntimeError> {
        let graph = RuntimeGraph::load_from_file(path)?;
        let id = uuid::Uuid::new_v4().to_string();
        // ... create ActiveStrategy
        Ok(id)
    }
    
    /// Start a strategy
    pub fn start_strategy(&mut self, id: &str) -> Result<(), RuntimeError> {
        // Find strategy, set status to Running
    }
    
    /// Stop a strategy
    pub fn stop_strategy(&mut self, id: &str) -> Result<(), RuntimeError> {
        // Find strategy, set status to Stopped
    }
    
    /// Main tick loop
    pub async fn run(&mut self) -> Result<(), RuntimeError> {
        let mut interval = tokio::time::interval(self.tick_interval);
        
        loop {
            interval.tick().await;
            self.tick().await?;
        }
    }
    
    /// Execute one tick
    pub async fn tick(&mut self) -> Result<Vec<OrderIntent>, RuntimeError> {
        let mut intents = Vec::new();
        
        for strategy in &self.active_strategies {
            if strategy.status != StrategyStatus::Running {
                continue;
            }
            
            // Get connector
            let connector = self.connectors.get(&strategy.connector)
                .ok_or(RuntimeError::ConnectorNotFound)?;
            
            // Create resolver from connector
            let resolver = ConnectorResolver::new(connector.clone());
            
            // Execute graph
            let outputs = self.vm
                .with_external_resolver(Arc::new(resolver))
                .execute(&strategy.graph)?;
            
            // Convert outputs to order intents
            // ...
            
            intents.extend(strategy_intents);
        }
        
        // Process intents
        for intent in &intents {
            self.process_intent(intent).await?;
        }
        
        Ok(intents)
    }
    
    /// Handle fill event
    pub fn on_fill(&mut self, event: FillEvent) {
        // Update order tracker
        // Update strategy state
        // Log the fill
    }
}
```

## Order Tracker

**File:** `src/orders/tracker.rs`

```rust
use std::collections::HashMap;
use chrono::{DateTime, Utc};

pub struct OrderTracker {
    orders: HashMap<String, TrackedOrder>,
    order_history: Vec<TrackedOrder>,
}

pub struct TrackedOrder {
    pub id: String,
    pub client_id: Option<String>,
    pub strategy_id: String,
    pub connector: String,
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
    pub fills: Vec<Fill>,
}

pub struct Fill {
    pub price: Decimal,
    pub quantity: Decimal,
    pub fee: Decimal,
    pub fee_asset: String,
    pub timestamp: DateTime<Utc>,
}

impl OrderTracker {
    pub fn new() -> Self {
        Self {
            orders: HashMap::new(),
            order_history: Vec::new(),
        }
    }
    
    pub fn track(&mut self, order: TrackedOrder) {
        self.orders.insert(order.id.clone(), order);
    }
    
    pub fn update(&mut self, id: &str, status: OrderStatus) {
        if let Some(order) = self.orders.get_mut(id) {
            order.status = status;
            order.updated_at = Utc::now();
        }
    }
    
    pub fn add_fill(&mut self, id: &str, fill: Fill) {
        if let Some(order) = self.orders.get_mut(id) {
            order.filled_quantity += fill.quantity;
            order.fills.push(fill);
            order.updated_at = Utc::now();
        }
    }
    
    pub fn get_open_orders(&self) -> Vec<&TrackedOrder> {
        self.orders.values()
            .filter(|o| matches!(o.status, OrderStatus::New | OrderStatus::PartiallyFilled))
            .collect()
    }
    
    pub fn get_by_strategy(&self, strategy_id: &str) -> Vec<&TrackedOrder> {
        self.orders.values()
            .filter(|o| o.strategy_id == strategy_id)
            .collect()
    }
}
```

## Backtesting Engine

**File:** `src/backtest/mod.rs`

```rust
use crate::runtime::TradingRuntime;
use crate::backtest::simulator::ExchangeSimulator;
use crate::backtest::metrics::PerformanceMetrics;

pub struct BacktestEngine {
    runtime: TradingRuntime,
    simulator: ExchangeSimulator,
    data: HistoricalData,
}

pub struct BacktestConfig {
    pub start_date: DateTime<Utc>,
    pub end_date: DateTime<Utc>,
    pub initial_balance: Decimal,
    pub fee_rate: Decimal,
    pub slippage_bps: Decimal,  // basis points
}

pub struct BacktestResult {
    pub metrics: PerformanceMetrics,
    pub trades: Vec<BacktestTrade>,
    pub equity_curve: Vec<(DateTime<Utc>, Decimal)>,
}

impl BacktestEngine {
    pub fn new(config: BacktestConfig) -> Self {
        let simulator = ExchangeSimulator::new(
            config.initial_balance,
            config.fee_rate,
            config.slippage_bps,
        );
        
        Self {
            runtime: TradingRuntime::new(),
            simulator,
            data: HistoricalData::empty(),
        }
    }
    
    /// Load historical data
    pub fn load_data(&mut self, path: &Path) -> Result<(), BacktestError> {
        self.data = HistoricalData::load_csv(path)?;
        Ok(())
    }
    
    /// Run backtest
    pub async fn run(&mut self, strategy_path: &Path) -> Result<BacktestResult, BacktestError> {
        // Load strategy
        let strategy_id = self.runtime.load_strategy(strategy_path)?;
        self.runtime.start_strategy(&strategy_id)?;
        
        // Replace real connector with simulator
        self.runtime.add_connector("backtest", Arc::new(self.simulator.clone()));
        
        let mut equity_curve = Vec::new();
        let mut trades = Vec::new();
        
        // Replay historical data
        for candle in self.data.candles() {
            // Update simulator with candle data
            self.simulator.update_price(candle.close);
            self.simulator.set_timestamp(candle.timestamp);
            
            // Run one tick
            let intents = self.runtime.tick().await?;
            
            // Record equity
            let equity = self.simulator.total_equity();
            equity_curve.push((candle.timestamp, equity));
            
            // Record trades
            for fill in self.simulator.get_fills() {
                trades.push(BacktestTrade::from_fill(fill));
            }
        }
        
        // Calculate metrics
        let metrics = PerformanceMetrics::calculate(&equity_curve, &trades);
        
        Ok(BacktestResult {
            metrics,
            trades,
            equity_curve,
        })
    }
}
```

### Performance Metrics

**File:** `src/backtest/metrics.rs`

```rust
pub struct PerformanceMetrics {
    pub total_pnl: Decimal,
    pub total_pnl_pct: Decimal,
    pub num_trades: u32,
    pub win_rate: Decimal,
    pub profit_factor: Decimal,
    pub max_drawdown: Decimal,
    pub max_drawdown_pct: Decimal,
    pub sharpe_ratio: f64,
    pub sortino_ratio: f64,
    pub avg_trade_pnl: Decimal,
    pub avg_win: Decimal,
    pub avg_loss: Decimal,
}

impl PerformanceMetrics {
    pub fn calculate(
        equity_curve: &[(DateTime<Utc>, Decimal)],
        trades: &[BacktestTrade],
    ) -> Self {
        // Calculate all metrics
        // ...
    }
    
    pub fn print_report(&self) {
        println!("═══════════════════════════════════════");
        println!("         BACKTEST RESULTS              ");
        println!("═══════════════════════════════════════");
        println!("Total PnL:        ${:.2} ({:.2}%)", self.total_pnl, self.total_pnl_pct);
        println!("Number of Trades: {}", self.num_trades);
        println!("Win Rate:         {:.2}%", self.win_rate * 100);
        println!("Profit Factor:    {:.2}", self.profit_factor);
        println!("Max Drawdown:     ${:.2} ({:.2}%)", self.max_drawdown, self.max_drawdown_pct);
        println!("Sharpe Ratio:     {:.2}", self.sharpe_ratio);
        println!("Sortino Ratio:    {:.2}", self.sortino_ratio);
        println!("═══════════════════════════════════════");
    }
}
```

## Paper Trading

**File:** `src/paper/mod.rs`

```rust
pub struct PaperTradingMode {
    balances: HashMap<String, Decimal>,
    positions: Vec<Position>,
    open_orders: Vec<Order>,
    fee_rate: Decimal,
}

impl PaperTradingMode {
    pub fn new(initial_balances: HashMap<String, Decimal>) -> Self {
        Self {
            balances: initial_balances,
            positions: Vec::new(),
            open_orders: Vec::new(),
            fee_rate: Decimal::from_str("0.001").unwrap(),  // 0.1%
        }
    }
    
    pub fn with_default_balances() -> Self {
        let mut balances = HashMap::new();
        balances.insert("USDT".to_string(), Decimal::from(10000));
        balances.insert("BTC".to_string(), Decimal::from(0));
        balances.insert("ETH".to_string(), Decimal::from(0));
        Self::new(balances)
    }
}

#[async_trait]
impl ConnectorBase for PaperTradingMode {
    fn name(&self) -> &str { "paper" }
    
    async fn get_ticker(&self, pair: &str) -> Result<Ticker, ConnectorError> {
        // Return simulated ticker based on real market data
    }
    
    async fn place_order(&self, order: &OrderRequest) -> Result<OrderResponse, ConnectorError> {
        // Simulate order execution with slippage
    }
    
    // ... implement other methods
}
```

## CLI Enhancement

**File:** `src/main.rs`

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "0-hummingbot")]
#[command(about = "High-frequency trading reimagined for machines")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Run a strategy in live mode
    Run {
        /// Path to strategy graph (.0 file)
        strategy: PathBuf,
        
        /// Connector to use (e.g., binance, okx)
        #[arg(short, long)]
        connector: String,
        
        /// Trading pair (e.g., BTC/USDT)
        #[arg(short, long)]
        pair: String,
        
        /// API key (or use env var)
        #[arg(long, env = "API_KEY")]
        api_key: Option<String>,
        
        /// API secret (or use env var)
        #[arg(long, env = "API_SECRET")]
        api_secret: Option<String>,
    },
    
    /// Run backtest on historical data
    Backtest {
        /// Path to strategy graph
        strategy: PathBuf,
        
        /// Start date (YYYY-MM-DD)
        #[arg(long)]
        from: String,
        
        /// End date (YYYY-MM-DD)
        #[arg(long)]
        to: String,
        
        /// Path to historical data
        #[arg(long)]
        data: PathBuf,
        
        /// Initial balance in USDT
        #[arg(long, default_value = "10000")]
        balance: Decimal,
    },
    
    /// Run in paper trading mode
    Paper {
        /// Path to strategy graph
        strategy: PathBuf,
        
        /// Connector to simulate
        #[arg(short, long)]
        connector: String,
        
        /// Trading pair
        #[arg(short, long)]
        pair: String,
        
        /// Initial balance in USDT
        #[arg(long, default_value = "10000")]
        balance: Decimal,
    },
    
    /// Inspect a strategy graph
    Inspect {
        /// Path to strategy graph
        strategy: PathBuf,
        
        /// Show detailed node information
        #[arg(short, long)]
        verbose: bool,
    },
    
    /// Verify strategy proofs
    Verify {
        /// Path to strategy graph
        strategy: PathBuf,
    },
    
    /// Compose multiple strategies
    Compose {
        /// Input strategy graphs
        inputs: Vec<PathBuf>,
        
        /// Output file
        #[arg(short, long)]
        output: PathBuf,
    },
    
    /// Show status of running strategies
    Status,
    
    /// Show open orders
    Orders,
    
    /// Show profit/loss report
    Pnl {
        /// Time period (today, week, month, all)
        #[arg(long, default_value = "today")]
        period: String,
    },
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Initialize logging
    tracing_subscriber::init();
    
    let cli = Cli::parse();
    
    match cli.command {
        Commands::Run { strategy, connector, pair, api_key, api_secret } => {
            run_live(strategy, connector, pair, api_key, api_secret).await?;
        }
        Commands::Backtest { strategy, from, to, data, balance } => {
            run_backtest(strategy, from, to, data, balance).await?;
        }
        Commands::Paper { strategy, connector, pair, balance } => {
            run_paper(strategy, connector, pair, balance).await?;
        }
        Commands::Inspect { strategy, verbose } => {
            inspect_strategy(strategy, verbose)?;
        }
        Commands::Verify { strategy } => {
            verify_strategy(strategy)?;
        }
        Commands::Compose { inputs, output } => {
            compose_strategies(inputs, output)?;
        }
        Commands::Status => {
            show_status().await?;
        }
        Commands::Orders => {
            show_orders().await?;
        }
        Commands::Pnl { period } => {
            show_pnl(period).await?;
        }
    }
    
    Ok(())
}
```

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-hummingbot
git checkout -b agent5/runtime

# Read existing code
cat src/main.rs
cat src/runtime.rs

# Create directory structure
mkdir -p src/runtime
mkdir -p src/backtest
mkdir -p src/paper
mkdir -p src/orders
mkdir -p src/logging
```

## Dependencies to Add (Cargo.toml)

```toml
# Async runtime
tokio = { version = "1.0", features = ["full"] }

# CLI
clap = { version = "4.0", features = ["derive", "env"] }

# Logging
tracing = "0.1"
tracing-subscriber = "0.3"

# Data handling
csv = "1.3"
parquet = "50.0"  # For efficient historical data

# Time
chrono = { version = "0.4", features = ["serde"] }

# UUID for strategy IDs
uuid = { version = "1.0", features = ["v4"] }

# Decimal
rust_decimal = "1.33"
```

## Commit Format

```
[Agent#5] feat: Implement TradingRuntime with tick loop
[Agent#5] feat: Add OrderTracker
[Agent#5] feat: Implement paper trading mode
[Agent#5] feat: Add backtesting engine
[Agent#5] feat: Enhance CLI with all commands
```

## Priority Order

1. TradingRuntime skeleton with tick loop
2. Order tracker
3. Paper trading (for safe testing)
4. CLI enhancement
5. Backtesting engine
6. Logging and monitoring

## Dependencies on Other Agents

- **Agent #2/#3**: Need ConnectorBase trait
- **Agent #4**: Need strategy graphs and PCO

## Handoff Points

When you complete:
- **TradingRuntime**: System is runnable end-to-end
- **Paper trading**: Safe testing environment
- **Backtesting**: Strategy validation capability

## Questions?

Document blockers in `BLOCKERS.md` in your branch.

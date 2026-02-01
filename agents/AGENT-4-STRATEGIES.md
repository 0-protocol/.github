# Agent #4: Strategy & Graph Developer

## Your Mission

You are responsible for implementing trading strategies as 0-lang graphs and building the Proof-Carrying Order system.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-hummingbot`

## Strategies to Implement

### Market Making Family

| Strategy | File | Description |
|----------|------|-------------|
| Pure Market Making | `pure_mm.0` | Single-pair, configurable spread |
| Avellaneda-Stoikov | `avellaneda_mm.0` | Optimal MM based on academic paper |
| Cross-Exchange MM | `cross_mm.0` | MM with hedge on another exchange |

### Arbitrage Family

| Strategy | File | Description |
|----------|------|-------------|
| Cross-Exchange Arb | `cross_arb.0` | Price differences between CEX |
| Spot-Perp Arb | `spot_perp_arb.0` | Funding rate arbitrage |
| DEX-CEX Arb | `dex_cex_arb.0` | On-chain vs off-chain |

### Execution Strategies

| Strategy | File | Description |
|----------|------|-------------|
| TWAP | `twap.0` | Time-weighted average price |
| VWAP | `vwap.0` | Volume-weighted average price |
| Iceberg | `iceberg.0` | Hidden large orders |

## File Structure

```
graphs/
├── strategies/
│   ├── market_making/
│   │   ├── pure_mm.0
│   │   ├── avellaneda_mm.0
│   │   └── cross_mm.0
│   ├── arbitrage/
│   │   ├── cross_arb.0
│   │   ├── spot_perp_arb.0
│   │   └── dex_cex_arb.0
│   └── execution/
│       ├── twap.0
│       ├── vwap.0
│       └── iceberg.0
├── components/              # Reusable subgraphs
│   ├── risk_check.0
│   ├── position_manager.0
│   └── order_executor.0
└── examples/
    └── simple_market_maker.0

src/pco/                     # Proof-Carrying Orders
├── mod.rs
├── builder.rs
└── verifier.rs

src/composer/                # Graph composition
├── mod.rs
└── linker.rs

schema/trading.capnp         # Trading domain types
```

## Strategy Graph Format

Each `.0` file should follow this structure:

```
# Strategy Name
#
# Description of what the strategy does.
#
# ============================================================================
# CONFIGURATION
# ============================================================================
#
# Parameters:
#   - param1: description (default: value)
#   - param2: description (default: value)
#
# ============================================================================
# GRAPH DEFINITION
# ============================================================================
#
# Graph {
#     name: "strategy_name",
#     version: 1,
#     
#     nodes: [
#         { id: sha256("node_name"), type: Constant|Operation|External|Branch, ... },
#         ...
#     ],
#     
#     entry_point: "first_node",
#     outputs: ["output_node"],
#     
#     proofs: [
#         HaltingProof { max_steps: N, fuel_budget: M }
#     ]
# }
#
# ============================================================================
# EXECUTION FLOW DIAGRAM
# ============================================================================
#
#   [ASCII diagram showing node flow]
#
# ============================================================================
# WHY 0-LANG IS BETTER
# ============================================================================
#
# [Explain the advantages for this specific strategy]
#
# ============================================================================

# Binary content placeholder
```

## Avellaneda-Stoikov Market Making

This is a priority strategy - implement the optimal market making algorithm.

**Reference:** "High-frequency trading in a limit order book" by Avellaneda & Stoikov (2008)

**Key Formula:**
```
reservation_price = mid_price - q * gamma * sigma^2 * T
spread = gamma * sigma^2 * T + (2/gamma) * ln(1 + gamma/k)

where:
- q = current inventory position
- gamma = risk aversion parameter
- sigma = volatility
- T = time horizon
- k = order arrival rate
```

**Graph Structure:**
```
┌──────────────────┐     ┌──────────────────┐
│  Get Mid Price   │     │  Get Position    │
│   (External)     │     │   (External)     │
└────────┬─────────┘     └────────┬─────────┘
         │                        │
         └────────────┬───────────┘
                      │
         ┌────────────▼───────────┐
         │ Calculate Reservation  │
         │      Price             │
         │  mid - q*γ*σ²*T       │
         └────────────┬───────────┘
                      │
         ┌────────────▼───────────┐
         │   Calculate Spread     │
         │   γ*σ²*T + (2/γ)*ln()  │
         └────────────┬───────────┘
                      │
         ┌────────────▼───────────┐
         │   Calculate Bid/Ask    │
         │  bid = res - spread/2  │
         │  ask = res + spread/2  │
         └────────────┬───────────┘
                      │
         ┌────────────▼───────────┐
         │     Place Orders       │
         │      (External)        │
         └────────────────────────┘
```

## Proof-Carrying Orders (PCO)

### PCO Structure

**File:** `src/pco/mod.rs`

```rust
use ed25519_dalek::{Keypair, PublicKey, Signature, Signer, Verifier};
use sha2::{Sha256, Digest};

/// A Proof-Carrying Order contains cryptographic proof of strategy intent
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProofCarryingOrder {
    /// The actual order to execute
    pub order: Order,
    
    /// Hash of the strategy graph that generated this order
    pub strategy_hash: [u8; 32],
    
    /// Hash of the market data inputs at decision time
    pub input_hash: [u8; 32],
    
    /// Execution trace: hashes of all nodes evaluated
    pub execution_trace: Vec<[u8; 32]>,
    
    /// Timestamp of order generation
    pub timestamp: u64,
    
    /// Agent's public key
    pub agent_pubkey: [u8; 32],
    
    /// Ed25519 signature over the above fields
    pub signature: [u8; 64],
}

impl ProofCarryingOrder {
    /// Create a new PCO from order and execution context
    pub fn new(
        order: Order,
        strategy_hash: [u8; 32],
        input_hash: [u8; 32],
        execution_trace: Vec<[u8; 32]>,
        keypair: &Keypair,
    ) -> Self {
        let timestamp = chrono::Utc::now().timestamp() as u64;
        let agent_pubkey = keypair.public.to_bytes();
        
        // Create message to sign
        let mut message = Vec::new();
        message.extend_from_slice(&order.to_bytes());
        message.extend_from_slice(&strategy_hash);
        message.extend_from_slice(&input_hash);
        for trace in &execution_trace {
            message.extend_from_slice(trace);
        }
        message.extend_from_slice(&timestamp.to_le_bytes());
        message.extend_from_slice(&agent_pubkey);
        
        let signature = keypair.sign(&message);
        
        Self {
            order,
            strategy_hash,
            input_hash,
            execution_trace,
            timestamp,
            agent_pubkey,
            signature: signature.to_bytes(),
        }
    }
    
    /// Verify the PCO signature
    pub fn verify(&self) -> Result<bool, PcoError> {
        let public_key = PublicKey::from_bytes(&self.agent_pubkey)?;
        let signature = Signature::from_bytes(&self.signature)?;
        
        let message = self.build_message();
        
        public_key.verify(&message, &signature)
            .map(|_| true)
            .map_err(|e| PcoError::InvalidSignature(e.to_string()))
    }
    
    /// Verify that the execution trace is valid for the strategy
    pub fn verify_execution(&self, strategy: &RuntimeGraph) -> Result<bool, PcoError> {
        // Verify that execution_trace matches valid path through strategy
        // ...
    }
}
```

### PCO Builder

**File:** `src/pco/builder.rs`

```rust
pub struct PcoBuilder {
    keypair: Keypair,
    execution_trace: Vec<[u8; 32]>,
}

impl PcoBuilder {
    pub fn new(keypair: Keypair) -> Self {
        Self {
            keypair,
            execution_trace: Vec::new(),
        }
    }
    
    /// Record a node evaluation
    pub fn record_node(&mut self, node_hash: [u8; 32]) {
        self.execution_trace.push(node_hash);
    }
    
    /// Build the final PCO
    pub fn build(
        self,
        order: Order,
        strategy_hash: [u8; 32],
        input_hash: [u8; 32],
    ) -> ProofCarryingOrder {
        ProofCarryingOrder::new(
            order,
            strategy_hash,
            input_hash,
            self.execution_trace,
            &self.keypair,
        )
    }
}
```

## Graph Composition

**File:** `src/composer/mod.rs`

```rust
/// Compose multiple graphs into one
pub struct GraphComposer {
    graphs: HashMap<[u8; 32], RuntimeGraph>,
}

impl GraphComposer {
    /// Import a subgraph by hash
    pub fn import(&mut self, graph: RuntimeGraph) -> [u8; 32] {
        let hash = graph.hash();
        self.graphs.insert(hash, graph);
        hash
    }
    
    /// Compose graphs by connecting outputs to inputs
    pub fn compose(
        &self,
        main_graph: &RuntimeGraph,
        connections: Vec<Connection>,
    ) -> Result<RuntimeGraph, ComposerError> {
        // Link subgraph outputs to main graph inputs
        // Verify no cycles are created
        // Return composed graph
    }
}

pub struct Connection {
    pub from_graph: [u8; 32],
    pub from_output: String,
    pub to_node: String,
    pub to_input: usize,
}
```

## Reusable Components

### Risk Check Component

**File:** `graphs/components/risk_check.0`

```
# Risk Check Component
#
# Reusable risk checking subgraph.
# Checks position limits, drawdown, and exposure.
#
# Inputs:
#   - current_position: Tensor<1>
#   - proposed_order_size: Tensor<1>
#   - account_balance: Tensor<1>
#
# Outputs:
#   - approved: Tensor<1> (0.0 or 1.0)
#   - adjusted_size: Tensor<1> (possibly reduced size)
```

### Position Manager Component

**File:** `graphs/components/position_manager.0`

```
# Position Manager Component
#
# Tracks and manages trading positions.
#
# Inputs:
#   - fill_event: Fill details
#   - current_state: Current position state
#
# Outputs:
#   - updated_state: New position state
#   - pnl: Realized P&L from this fill
```

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-hummingbot
git checkout -b agent4/strategies

# Read existing strategies
cat graphs/strategies/market_making.0
cat graphs/strategies/arbitrage.0
cat graphs/strategies/grid_trading.0
```

## Available 0-lang Operations

Already implemented (you can use these):
- Math: `Add`, `Sub`, `Mul`, `Div`, `Matmul`
- Comparisons: `Eq`, `Gt`, `Lt`, `Gte`, `Lte`
- Reductions: `Sum`, `Mean`, `Argmax`, `Min`, `Max`
- Utilities: `Abs`, `Neg`, `Clamp`, `Concat`, `Reshape`
- Activations: `Softmax`, `Relu`, `Sigmoid`, `Tanh`

## Dependencies to Add (Cargo.toml)

```toml
ed25519-dalek = "2.0"
sha2 = "0.10"
serde = { version = "1.0", features = ["derive"] }
```

## Commit Format

```
[Agent#4] feat: Implement Avellaneda-Stoikov strategy
[Agent#4] feat: Add PCO builder and verifier
[Agent#4] feat: Implement TWAP execution strategy
[Agent#4] feat: Add graph composition system
```

## Priority Order

1. Enhance existing strategies (add more detail, formulas)
2. Implement Avellaneda-Stoikov MM
3. Implement TWAP/VWAP execution
4. Build PCO system
5. Graph composition
6. Reusable components

## Dependencies on Other Agents

- **Agent #1**: Use new operations (Min, Max, Clamp already added)
- **Agent #2/#3**: Need connector interface for External nodes

## Handoff Points

When you complete:
- **PCO system**: Agent #5 can integrate into order flow
- **Strategy graphs**: Agent #5 can test with runtime
- **Composition**: Enables complex multi-component strategies

## Questions?

Document blockers in `BLOCKERS.md` in your branch.

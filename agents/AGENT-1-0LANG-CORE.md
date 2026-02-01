# Agent #1: 0-lang Core Developer

## Your Mission

You are responsible for enhancing 0-lang core capabilities for trading applications.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-lang`

## Features to Implement

### 1. StringTensor (Priority: P0 - Blocks other agents)

Add string tensor type for handling API responses, trading pairs, etc.

**File:** `src/tensor.rs`

```rust
pub enum TensorData {
    Float(Vec<f32>),
    String(Vec<String>),  // NEW
    Decimal(Vec<Decimal>), // NEW (see #3)
}

pub struct Tensor {
    pub shape: Vec<u32>,
    pub data: TensorData,
    pub confidence: f32,
}
```

### 2. JSON Operations (Priority: P0)

Add operations for parsing and extracting JSON data from API responses.

**Files:** `src/graph.rs`, `src/vm.rs`, `schema/zero.capnp`

New operations:
- `Op::JsonParse` - Parse JSON string into structured tensor
- `Op::JsonGet` - Extract value by key path (e.g., "data.price")
- `Op::JsonArray` - Extract array elements

### 3. DecimalTensor (Priority: P1)

Add decimal tensor for financial precision calculations.

**File:** `src/tensor.rs`

```rust
use rust_decimal::Decimal;

// Add to Cargo.toml: rust_decimal = "1.33"
```

### 4. Async External Execution (Priority: P1 - Blocks Agent #3)

Modify VM to support async/await for External nodes.

**File:** `src/vm.rs`

```rust
impl VM {
    // Change from sync to async
    pub async fn execute(&mut self, graph: &RuntimeGraph) -> Result<HashMap<NodeHash, Tensor>, VMError>;
    
    // Add parallel execution
    pub async fn execute_parallel(&mut self, nodes: Vec<&RuntimeNode>) -> Result<Vec<Tensor>, VMError>;
}
```

### 5. State Persistence (Priority: P2)

Add StateNode type for mutable state across executions (positions, balances).

**File:** `src/graph.rs`

```rust
pub enum RuntimeNode {
    Constant(Tensor),
    Operation { op: Op, inputs: Vec<NodeHash> },
    Branch { ... },
    External { uri: String, inputs: Vec<NodeHash> },
    State { key: String, default: Tensor },  // NEW
}
```

### 6. Event System (Priority: P2)

Create event system for tick-based execution.

**New File:** `src/events.rs`

```rust
pub enum TradingEvent {
    Tick { timestamp: u64 },
    Fill { order_id: String, price: Decimal, quantity: Decimal },
    OrderUpdate { order_id: String, status: OrderStatus },
    PriceUpdate { pair: String, price: Decimal },
}

pub trait EventHandler {
    fn on_event(&mut self, event: TradingEvent) -> Vec<NodeHash>;
}
```

## Files You Own

```
0-lang/
├── src/
│   ├── tensor.rs        # StringTensor, DecimalTensor
│   ├── vm.rs            # Async execution, state handling
│   ├── graph.rs         # New node types, new ops
│   ├── verify.rs        # Update for new ops
│   ├── lib.rs           # Export new types
│   ├── events.rs        # NEW: Event system
│   └── stdlib/          # NEW: Standard library
│       ├── mod.rs
│       └── json.rs      # JSON operations
├── schema/
│   └── zero.capnp       # Schema updates
└── tests/
    ├── tensor_tests.rs  # Test new tensor types
    └── async_tests.rs   # Test async execution
```

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-lang
git checkout -b agent1/0lang-core

# First, read the existing code
cat src/tensor.rs
cat src/vm.rs
cat src/graph.rs
```

## Commit Format

```
[Agent#1] feat: Add StringTensor type
[Agent#1] feat: Implement JsonParse operation
[Agent#1] refactor: Make VM execution async
```

## Success Criteria

1. All existing tests pass (`cargo test`)
2. New tests for each feature
3. `cargo build` succeeds
4. Documentation for new APIs

## Dependencies to Add (Cargo.toml)

```toml
rust_decimal = "1.33"
serde_json = "1.0"
tokio = { version = "1.0", features = ["full"] }
async-trait = "0.1"
```

## Handoff Points

When you complete:
- **StringTensor**: Notify Agent #2 and #3 (they need it for API responses)
- **Async execution**: Notify Agent #3 (DEX needs async for on-chain calls)
- **DecimalTensor**: Notify Agent #4 (strategies need precise calculations)

## Current State

Already implemented (you added these):
- `Op::Gte`, `Op::Lte` - Comparison operators
- `Op::Min`, `Op::Max` - Min/max operations
- `Op::Abs`, `Op::Neg` - Math operations
- `Op::Clamp` - Clamp to range

## Questions?

If you encounter blocking issues or need clarification, document them in a `BLOCKERS.md` file in your branch.

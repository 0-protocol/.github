# Agent #6: 0-lang Extensions for Assistant

## Your Mission

You are responsible for extending 0-lang with new features required for AI assistant applications. These extensions will be used by 0-openclaw and future assistant projects.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-lang`

## Features to Implement

### 1. StreamTensor (Priority: P0 - Blocks Agent #7, #8)

Add streaming tensor support for handling message streams (WebSocket, SSE).

**File:** `src/tensor.rs`

```rust
pub enum TensorData {
    Float(Vec<f32>),
    String(Vec<String>),
    Decimal(Vec<Decimal>),
    Stream(StreamHandle),  // NEW
}

/// Handle to a streaming data source
pub struct StreamHandle {
    pub id: u64,
    pub source_type: StreamSource,
    pub buffer: Arc<RwLock<VecDeque<Tensor>>>,
}

pub enum StreamSource {
    WebSocket { url: String },
    Channel { channel_id: String },
    Event { event_type: String },
}
```

### 2. PermissionNode (Priority: P0 - Blocks Agent #7)

Add confidence-based permission checking node type.

**File:** `src/graph.rs`

```rust
pub enum RuntimeNode {
    Constant(Tensor),
    Operation { op: Op, inputs: Vec<NodeHash> },
    Branch { condition: NodeHash, true_branch: NodeHash, false_branch: NodeHash },
    External { uri: String, inputs: Vec<NodeHash> },
    State { key: String, default: Tensor },
    Permission {                          // NEW
        subject: NodeHash,                // Who is requesting
        action: NodeHash,                 // What they want to do
        threshold: f32,                   // Minimum confidence required
        fallback: NodeHash,               // What to do if denied
    },
}
```

### 3. RouteNode (Priority: P0 - Blocks Agent #7)

Add multi-path routing node with priority ordering.

**File:** `src/graph.rs`

```rust
pub struct RouteNode {
    /// Input to route
    pub input: NodeHash,
    
    /// Routes in priority order (first matching wins)
    pub routes: Vec<Route>,
    
    /// Default route if none match
    pub default: NodeHash,
}

pub struct Route {
    /// Condition graph (must output confidence > threshold)
    pub condition: NodeHash,
    
    /// Minimum confidence to take this route
    pub threshold: f32,
    
    /// Target node if route matches
    pub target: NodeHash,
    
    /// Route metadata (for tracing)
    pub metadata: RouteMetadata,
}

pub struct RouteMetadata {
    pub name: String,
    pub description: String,
}
```

### 4. TimerNode (Priority: P1 - Blocks Agent #9)

Add scheduled execution support for cron-like behavior.

**File:** `src/graph.rs`

```rust
pub struct TimerNode {
    /// Cron expression (e.g., "*/5 * * * *" for every 5 minutes)
    pub schedule: String,
    
    /// Graph to execute when timer fires
    pub target: NodeHash,
    
    /// Maximum concurrent executions
    pub max_concurrent: u32,
    
    /// What to do if execution is still running when timer fires
    pub overlap_policy: OverlapPolicy,
}

pub enum OverlapPolicy {
    Skip,      // Skip this execution
    Queue,     // Queue for later
    Parallel,  // Run in parallel
}
```

### 5. Confidence Operations (Priority: P0)

Add operations for manipulating confidence scores.

**File:** `src/graph.rs`

```rust
pub enum Op {
    // ... existing ops ...
    
    // Confidence operations
    ConfidenceCombine,    // Combine multiple confidence scores
    ConfidenceThreshold,  // Output 1.0 if above threshold, 0.0 otherwise
    ConfidenceDecay,      // Reduce confidence over time
    ConfidenceBoost,      // Increase confidence based on history
}
```

**File:** `src/vm.rs`

```rust
impl VM {
    fn execute_confidence_combine(&self, inputs: &[Tensor]) -> Result<Tensor, VMError> {
        // Combine confidence scores using configurable strategy
        // Options: min, max, product, average, weighted
        let scores: Vec<f32> = inputs.iter()
            .map(|t| t.as_scalar())
            .collect::<Result<_, _>>()?;
        
        // Default: geometric mean (product ^ 1/n)
        let product: f32 = scores.iter().product();
        let combined = product.powf(1.0 / scores.len() as f32);
        
        Ok(Tensor::scalar(combined))
    }
    
    fn execute_confidence_threshold(&self, input: &Tensor, threshold: f32) -> Result<Tensor, VMError> {
        let score = input.as_scalar()?;
        Ok(Tensor::scalar(if score >= threshold { 1.0 } else { 0.0 }))
    }
}
```

### 6. Trace Generation (Priority: P0 - Blocks Agent #7)

Add execution trace generation for proof-carrying actions.

**File:** `src/vm.rs`

```rust
pub struct ExecutionTrace {
    /// Ordered list of node hashes executed
    pub nodes: Vec<[u8; 32]>,
    
    /// Timestamp of each execution
    pub timestamps: Vec<u64>,
    
    /// Confidence score at each step
    pub confidences: Vec<f32>,
}

impl VM {
    /// Execute graph and return result with trace
    pub async fn execute_with_trace(
        &mut self, 
        graph: &RuntimeGraph
    ) -> Result<(HashMap<NodeHash, Tensor>, ExecutionTrace), VMError> {
        let mut trace = ExecutionTrace::new();
        
        // Execute graph, recording each node
        for node in self.execution_order(graph)? {
            let start = std::time::Instant::now();
            
            // Execute node
            let result = self.execute_node(node)?;
            
            // Record in trace
            trace.nodes.push(node.hash());
            trace.timestamps.push(chrono::Utc::now().timestamp_millis() as u64);
            trace.confidences.push(result.confidence().unwrap_or(1.0));
        }
        
        Ok((results, trace))
    }
}
```

## File Structure

```
0-lang/src/
├── tensor.rs           # Add StreamTensor
├── graph.rs            # Add PermissionNode, RouteNode, TimerNode
├── vm.rs               # Add trace generation, confidence ops
├── stream.rs           # NEW: Stream handling
├── permission.rs       # NEW: Permission system
├── route.rs            # NEW: Routing system
└── timer.rs            # NEW: Timer system
```

## New Files to Create

### src/stream.rs

```rust
use std::collections::VecDeque;
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct StreamManager {
    streams: HashMap<u64, StreamHandle>,
    next_id: AtomicU64,
}

impl StreamManager {
    pub fn new() -> Self {
        Self {
            streams: HashMap::new(),
            next_id: AtomicU64::new(0),
        }
    }
    
    /// Create a new stream from a WebSocket
    pub async fn from_websocket(&mut self, url: &str) -> Result<StreamHandle, StreamError> {
        let id = self.next_id.fetch_add(1, Ordering::SeqCst);
        let buffer = Arc::new(RwLock::new(VecDeque::new()));
        
        // Spawn WebSocket listener
        let buffer_clone = buffer.clone();
        tokio::spawn(async move {
            // Connect and push to buffer
        });
        
        let handle = StreamHandle {
            id,
            source_type: StreamSource::WebSocket { url: url.to_string() },
            buffer,
        };
        
        self.streams.insert(id, handle.clone());
        Ok(handle)
    }
    
    /// Read next item from stream
    pub async fn read(&self, handle: &StreamHandle) -> Option<Tensor> {
        let mut buffer = handle.buffer.write().await;
        buffer.pop_front()
    }
}
```

### src/permission.rs

```rust
use crate::types::Confidence;

/// Permission evaluation result
pub struct PermissionResult {
    pub allowed: bool,
    pub confidence: Confidence,
    pub reason: String,
}

/// Permission policy configuration
pub struct PermissionPolicy {
    /// Minimum confidence required
    pub threshold: f32,
    
    /// How to combine multiple permission checks
    pub combination: CombinationStrategy,
    
    /// Whether to log permission decisions
    pub audit: bool,
}

pub enum CombinationStrategy {
    All,      // All checks must pass
    Any,      // Any check can pass
    Majority, // More than half must pass
    Weighted(Vec<f32>), // Weighted combination
}

pub fn evaluate_permission(
    subject: &Tensor,
    action: &Tensor,
    policy: &PermissionPolicy,
) -> PermissionResult {
    // Evaluate permission based on policy
    // Returns confidence score
}
```

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-lang
git checkout -b agent6/assistant-extensions

# Read existing code
cat src/tensor.rs
cat src/graph.rs
cat src/vm.rs

# Create new files
touch src/stream.rs
touch src/permission.rs
touch src/route.rs
touch src/timer.rs
```

## Dependencies to Add (Cargo.toml)

```toml
# WebSocket for streaming
tokio-tungstenite = { version = "0.21", features = ["native-tls"] }

# Cron parsing for TimerNode
cron = "0.12"

# Time utilities
chrono = "0.4"
```

## Commit Format

```
[Agent#6] feat: Add StreamTensor for message streams
[Agent#6] feat: Implement PermissionNode with confidence threshold
[Agent#6] feat: Add RouteNode for multi-path routing
[Agent#6] feat: Implement TimerNode for scheduled execution
[Agent#6] feat: Add confidence operations (combine, threshold, decay)
[Agent#6] feat: Add execution trace generation
```

## Testing

Create tests for each new feature:

```rust
// tests/stream_tests.rs
#[tokio::test]
async fn test_stream_tensor() {
    let mut manager = StreamManager::new();
    // Test stream creation and reading
}

// tests/permission_tests.rs
#[test]
fn test_permission_evaluation() {
    let policy = PermissionPolicy {
        threshold: 0.8,
        combination: CombinationStrategy::All,
        audit: true,
    };
    // Test permission checks
}

// tests/route_tests.rs
#[test]
fn test_multi_path_routing() {
    // Test route selection
}
```

## Success Criteria

1. All existing tests pass (`cargo test`)
2. New tests for each feature
3. `cargo build` succeeds
4. Documentation for new APIs
5. Stream handling works with WebSocket
6. Permission system evaluates correctly
7. Routing selects correct paths
8. Timer system fires on schedule

## Dependencies on Other Agents

- **Agent #1**: Build on existing StringTensor, DecimalTensor work
- **Agent #0**: Need 0-openclaw repository ready for integration testing

## Handoff Points

When you complete:
- **StreamTensor**: Notify Agent #8 (channel connectors need it)
- **PermissionNode**: Notify Agent #7 (gateway needs it)
- **RouteNode**: Notify Agent #7 (gateway routing needs it)
- **TimerNode**: Notify Agent #9 (skills may use scheduled execution)
- **Trace generation**: Notify Agent #7 (proof-carrying actions need it)

## Questions?

Document blockers in `BLOCKERS.md` in your branch.

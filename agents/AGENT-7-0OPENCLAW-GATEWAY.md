# Agent #7: 0-openclaw Gateway & Sessions

## Your Mission

You are responsible for building the core Gateway - the control plane of 0-openclaw. This includes session management, message routing, and the proof-carrying action system.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-openclaw`

## Components to Build

| Component | Priority | Description |
|-----------|----------|-------------|
| Gateway struct | P0 | Main control plane |
| Session Manager | P0 | Graph-based session state |
| Router | P0 | Content-addressed message routing |
| Proof Generator | P0 | Create Proof-Carrying Actions |
| WebSocket Server | P1 | Gateway API |
| Event Bus | P1 | Internal event distribution |

## File Structure

```
src/gateway/
├── mod.rs              # Gateway struct, re-exports
├── session.rs          # Session management
├── router.rs           # Message routing
├── proof.rs            # Proof-Carrying Action generation
├── server.rs           # WebSocket server
├── events.rs           # Event bus
└── config.rs           # Gateway configuration

graphs/core/
├── router.0            # Main routing graph
├── session.0           # Session management graph
└── permission.0        # Permission checking graph
```

## Gateway Implementation

### Core Gateway Struct

**File:** `src/gateway/mod.rs`

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;
use zerolang::{VM, RuntimeGraph};
use crate::types::{ContentHash, ProofCarryingAction, IncomingMessage};
use crate::channels::Channel;
use crate::skills::SkillRegistry;

pub struct Gateway {
    /// The 0-lang VM for graph execution
    vm: VM,
    
    /// Main routing graph
    router_graph: RuntimeGraph,
    
    /// Session manager
    sessions: Arc<RwLock<SessionManager>>,
    
    /// Registered channels
    channels: HashMap<String, Arc<dyn Channel>>,
    
    /// Skill registry
    skills: Arc<SkillRegistry>,
    
    /// Proof generator
    proof_generator: ProofGenerator,
    
    /// Event bus for internal communication
    event_bus: EventBus,
    
    /// Gateway configuration
    config: GatewayConfig,
}

impl Gateway {
    pub fn new(config: GatewayConfig) -> Result<Self, GatewayError> {
        let router_graph = RuntimeGraph::load_from_file(&config.router_graph_path)?;
        
        Ok(Self {
            vm: VM::new(),
            router_graph,
            sessions: Arc::new(RwLock::new(SessionManager::new())),
            channels: HashMap::new(),
            skills: Arc::new(SkillRegistry::new()),
            proof_generator: ProofGenerator::new(&config.keypair_path)?,
            event_bus: EventBus::new(),
            config,
        })
    }
    
    /// Register a channel
    pub fn register_channel(&mut self, channel: Arc<dyn Channel>) {
        self.channels.insert(channel.name().to_string(), channel);
    }
    
    /// Process incoming message - core loop
    pub async fn process_message(
        &self, 
        message: IncomingMessage
    ) -> Result<ProofCarryingAction, GatewayError> {
        // 1. Get or create session
        let session = self.sessions.write().await
            .get_or_create(&message.channel_id, &message.sender_id)?;
        
        // 2. Execute routing graph with trace
        let (route_result, route_trace) = self.vm
            .execute_with_trace(&self.router_graph)
            .await?;
        
        // 3. Determine target skill
        let skill_hash = self.extract_skill_hash(&route_result)?;
        
        // 4. Execute skill graph with trace
        let skill_graph = self.skills.get(skill_hash)?;
        let (skill_result, skill_trace) = self.vm
            .execute_with_trace(&skill_graph)
            .await?;
        
        // 5. Build action from result
        let action = self.build_action(&skill_result)?;
        
        // 6. Generate proof-carrying action
        let pca = self.proof_generator.generate(
            action,
            session.hash(),
            message.id,
            vec![route_trace, skill_trace],
        )?;
        
        // 7. Update session state
        self.sessions.write().await.update(&session.id, &pca)?;
        
        Ok(pca)
    }
    
    /// Start the gateway
    pub async fn run(&self) -> Result<(), GatewayError> {
        // Start channel listeners
        for (name, channel) in &self.channels {
            let gateway = self.clone();
            let channel = channel.clone();
            
            tokio::spawn(async move {
                loop {
                    match channel.receive().await {
                        Ok(message) => {
                            match gateway.process_message(message).await {
                                Ok(pca) => {
                                    if let Err(e) = gateway.execute_action(&pca).await {
                                        tracing::error!("Action execution failed: {}", e);
                                    }
                                }
                                Err(e) => {
                                    tracing::error!("Message processing failed: {}", e);
                                }
                            }
                        }
                        Err(e) => {
                            tracing::error!("Channel receive error: {}", e);
                        }
                    }
                }
            });
        }
        
        // Start WebSocket server
        self.start_server().await
    }
}
```

### Session Manager

**File:** `src/gateway/session.rs`

```rust
use std::collections::HashMap;
use crate::types::{ContentHash, Confidence, ProofCarryingAction};

pub struct SessionManager {
    sessions: HashMap<ContentHash, Session>,
}

pub struct Session {
    pub id: ContentHash,
    pub channel_id: String,
    pub user_id: String,
    pub state: SessionState,
    pub history: Vec<ContentHash>,  // Hashes of past actions
    pub trust_score: Confidence,    // Accumulated trust
    pub created_at: u64,
    pub last_activity: u64,
}

pub struct SessionState {
    /// Serialized state tensor
    pub data: Vec<u8>,
    
    /// State version for conflict detection
    pub version: u64,
    
    /// Hash of current state
    pub hash: ContentHash,
}

impl SessionManager {
    pub fn new() -> Self {
        Self {
            sessions: HashMap::new(),
        }
    }
    
    /// Get existing session or create new one
    pub fn get_or_create(
        &mut self,
        channel_id: &str,
        user_id: &str,
    ) -> Result<&Session, SessionError> {
        let session_key = Self::session_key(channel_id, user_id);
        
        if !self.sessions.contains_key(&session_key) {
            let session = Session {
                id: session_key,
                channel_id: channel_id.to_string(),
                user_id: user_id.to_string(),
                state: SessionState::default(),
                history: Vec::new(),
                trust_score: Confidence::new(0.5),  // Start neutral
                created_at: chrono::Utc::now().timestamp_millis() as u64,
                last_activity: chrono::Utc::now().timestamp_millis() as u64,
            };
            self.sessions.insert(session_key, session);
        }
        
        Ok(self.sessions.get(&session_key).unwrap())
    }
    
    /// Update session after action
    pub fn update(
        &mut self,
        session_id: &ContentHash,
        action: &ProofCarryingAction,
    ) -> Result<(), SessionError> {
        let session = self.sessions.get_mut(session_id)
            .ok_or(SessionError::NotFound)?;
        
        // Update history
        session.history.push(action.input_hash);
        
        // Update trust score based on action confidence
        session.trust_score = Self::update_trust(
            session.trust_score,
            action.confidence,
        );
        
        session.last_activity = chrono::Utc::now().timestamp_millis() as u64;
        
        Ok(())
    }
    
    /// Calculate new trust score
    fn update_trust(current: Confidence, action_confidence: Confidence) -> Confidence {
        // Exponential moving average
        let alpha = 0.1;
        let new_value = (1.0 - alpha) * current.value() + alpha * action_confidence.value();
        Confidence::new(new_value)
    }
    
    /// Generate session key from channel and user
    fn session_key(channel_id: &str, user_id: &str) -> ContentHash {
        ContentHash::from_bytes(format!("{}:{}", channel_id, user_id).as_bytes())
    }
}
```

### Proof Generator

**File:** `src/gateway/proof.rs`

```rust
use ed25519_dalek::{SigningKey, Signature, Signer};
use crate::types::{ContentHash, Action, ProofCarryingAction, Confidence};
use zerolang::ExecutionTrace;

pub struct ProofGenerator {
    signing_key: SigningKey,
}

impl ProofGenerator {
    pub fn new(keypair_path: &str) -> Result<Self, ProofError> {
        let key_bytes = std::fs::read(keypair_path)?;
        let signing_key = SigningKey::from_bytes(&key_bytes.try_into()
            .map_err(|_| ProofError::InvalidKey)?);
        
        Ok(Self { signing_key })
    }
    
    /// Generate a Proof-Carrying Action
    pub fn generate(
        &self,
        action: Action,
        session_hash: ContentHash,
        input_hash: ContentHash,
        traces: Vec<ExecutionTrace>,
    ) -> Result<ProofCarryingAction, ProofError> {
        let timestamp = chrono::Utc::now().timestamp_millis() as u64;
        
        // Combine all traces
        let execution_trace: Vec<ContentHash> = traces
            .into_iter()
            .flat_map(|t| t.nodes.into_iter().map(ContentHash))
            .collect();
        
        // Calculate combined confidence
        let confidence = self.calculate_confidence(&execution_trace);
        
        // Build message to sign
        let message = self.build_sign_message(
            &action,
            &session_hash,
            &input_hash,
            &execution_trace,
            confidence,
            timestamp,
        );
        
        // Sign
        let signature: Signature = self.signing_key.sign(&message);
        
        Ok(ProofCarryingAction {
            action,
            session_hash,
            input_hash,
            execution_trace,
            confidence,
            signature: signature.to_bytes(),
            timestamp,
        })
    }
    
    /// Verify a Proof-Carrying Action
    pub fn verify(&self, pca: &ProofCarryingAction) -> Result<bool, ProofError> {
        let verifying_key = self.signing_key.verifying_key();
        let message = self.build_sign_message(
            &pca.action,
            &pca.session_hash,
            &pca.input_hash,
            &pca.execution_trace,
            pca.confidence,
            pca.timestamp,
        );
        
        let signature = Signature::from_bytes(&pca.signature);
        verifying_key.verify(&message, &signature)
            .map(|_| true)
            .map_err(|e| ProofError::InvalidSignature(e.to_string()))
    }
    
    fn build_sign_message(
        &self,
        action: &Action,
        session_hash: &ContentHash,
        input_hash: &ContentHash,
        execution_trace: &[ContentHash],
        confidence: Confidence,
        timestamp: u64,
    ) -> Vec<u8> {
        let mut message = Vec::new();
        
        // Serialize action
        message.extend_from_slice(&serde_json::to_vec(action).unwrap());
        message.extend_from_slice(&session_hash.0);
        message.extend_from_slice(&input_hash.0);
        
        for trace_hash in execution_trace {
            message.extend_from_slice(&trace_hash.0);
        }
        
        message.extend_from_slice(&confidence.value().to_le_bytes());
        message.extend_from_slice(&timestamp.to_le_bytes());
        
        message
    }
    
    fn calculate_confidence(&self, trace: &[ContentHash]) -> Confidence {
        // Start with high confidence, decay based on trace length
        // More complex paths = slightly lower confidence
        let base = 0.99;
        let decay = 0.001 * trace.len() as f32;
        Confidence::new((base - decay).max(0.5))
    }
}
```

### Message Router

**File:** `src/gateway/router.rs`

```rust
use zerolang::{VM, RuntimeGraph, Tensor};
use crate::types::{ContentHash, IncomingMessage};

pub struct Router {
    /// The routing graph
    graph: RuntimeGraph,
    
    /// Cached routes for fast lookup
    route_cache: HashMap<ContentHash, ContentHash>,
}

impl Router {
    pub fn new(graph: RuntimeGraph) -> Self {
        Self {
            graph,
            route_cache: HashMap::new(),
        }
    }
    
    /// Route a message to a skill
    pub async fn route(
        &self,
        vm: &mut VM,
        message: &IncomingMessage,
    ) -> Result<(ContentHash, ExecutionTrace), RouterError> {
        // Check cache first
        let message_pattern = Self::extract_pattern(message);
        if let Some(skill_hash) = self.route_cache.get(&message_pattern) {
            return Ok((*skill_hash, ExecutionTrace::cached()));
        }
        
        // Prepare input tensor
        let input = Self::message_to_tensor(message)?;
        
        // Execute routing graph
        let (outputs, trace) = vm.execute_with_trace(&self.graph).await?;
        
        // Extract skill hash from output
        let skill_hash = Self::extract_skill_hash(&outputs)?;
        
        Ok((skill_hash, trace))
    }
    
    fn message_to_tensor(message: &IncomingMessage) -> Result<Tensor, RouterError> {
        // Convert message to tensor for graph input
        let content_bytes = message.content.as_bytes();
        Ok(Tensor::from_string(&message.content))
    }
    
    fn extract_pattern(message: &IncomingMessage) -> ContentHash {
        // Extract pattern for caching (e.g., command prefix)
        if message.content.starts_with('/') {
            let command = message.content.split_whitespace().next().unwrap_or("");
            ContentHash::from_bytes(command.as_bytes())
        } else {
            ContentHash::from_bytes(b"default")
        }
    }
    
    fn extract_skill_hash(outputs: &HashMap<NodeHash, Tensor>) -> Result<ContentHash, RouterError> {
        // Extract skill hash from graph outputs
        // Implementation depends on graph structure
    }
}
```

### WebSocket Server

**File:** `src/gateway/server.rs`

```rust
use axum::{
    extract::ws::{WebSocket, WebSocketUpgrade, Message},
    routing::get,
    Router,
};
use std::sync::Arc;
use tokio::sync::broadcast;

pub struct GatewayServer {
    gateway: Arc<Gateway>,
    broadcast_tx: broadcast::Sender<ServerMessage>,
}

#[derive(Clone, serde::Serialize)]
pub enum ServerMessage {
    ActionExecuted(ProofCarryingAction),
    SessionUpdated { session_id: ContentHash },
    Error { message: String },
}

impl GatewayServer {
    pub fn new(gateway: Arc<Gateway>) -> Self {
        let (broadcast_tx, _) = broadcast::channel(100);
        Self {
            gateway,
            broadcast_tx,
        }
    }
    
    pub async fn start(&self, port: u16) -> Result<(), ServerError> {
        let app = Router::new()
            .route("/ws", get(Self::ws_handler))
            .route("/health", get(Self::health))
            .route("/sessions", get(Self::list_sessions))
            .with_state(self.clone());
        
        let addr = format!("127.0.0.1:{}", port);
        tracing::info!("Gateway server listening on {}", addr);
        
        axum::serve(
            tokio::net::TcpListener::bind(&addr).await?,
            app
        ).await?;
        
        Ok(())
    }
    
    async fn ws_handler(
        ws: WebSocketUpgrade,
        state: axum::extract::State<Arc<GatewayServer>>,
    ) -> impl axum::response::IntoResponse {
        ws.on_upgrade(move |socket| Self::handle_socket(socket, state.0))
    }
    
    async fn handle_socket(mut socket: WebSocket, server: Arc<GatewayServer>) {
        let mut rx = server.broadcast_tx.subscribe();
        
        loop {
            tokio::select! {
                // Handle incoming messages
                Some(msg) = socket.recv() => {
                    match msg {
                        Ok(Message::Text(text)) => {
                            // Handle client request
                        }
                        Ok(Message::Close(_)) => break,
                        _ => {}
                    }
                }
                // Broadcast server messages
                Ok(server_msg) = rx.recv() => {
                    let json = serde_json::to_string(&server_msg).unwrap();
                    if socket.send(Message::Text(json)).await.is_err() {
                        break;
                    }
                }
            }
        }
    }
    
    async fn health() -> &'static str {
        "ok"
    }
    
    async fn list_sessions(
        state: axum::extract::State<Arc<GatewayServer>>,
    ) -> axum::Json<Vec<SessionInfo>> {
        // Return session list
    }
}
```

## Graphs to Create

### graphs/core/router.0

```
# Main Router Graph
#
# Routes incoming messages to appropriate skills.
#
# Inputs:
#   - message: StringTensor containing message content
#   - sender: StringTensor containing sender ID
#   - channel: StringTensor containing channel ID
#
# Outputs:
#   - skill_hash: Tensor containing target skill hash
#   - confidence: Confidence score for routing decision

Graph {
    name: "main_router",
    version: 1,
    
    nodes: [
        # Input nodes
        { id: "message", type: External, uri: "input://message" },
        { id: "sender", type: External, uri: "input://sender" },
        { id: "channel", type: External, uri: "input://channel" },
        
        # Command detection
        { id: "is_command", type: Operation, op: StartsWith, inputs: ["message", "/"] },
        
        # Route based on command
        { id: "route", type: Route, input: "message", routes: [
            { condition: "is_command", threshold: 0.9, target: "command_handler" },
            { condition: "default", threshold: 0.5, target: "conversation_handler" },
        ]},
        
        # Output
        { id: "skill_hash", type: Operation, op: RouteOutput, inputs: ["route"] },
    ],
    
    entry_point: "message",
    outputs: ["skill_hash"],
}
```

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-openclaw
git checkout -b agent7/gateway

# Wait for Agent #0 to create structure, then:
cat src/lib.rs
cat src/types.rs

# Create gateway module
mkdir -p src/gateway
touch src/gateway/mod.rs
touch src/gateway/session.rs
touch src/gateway/router.rs
touch src/gateway/proof.rs
touch src/gateway/server.rs
touch src/gateway/events.rs
touch src/gateway/config.rs
```

## Commit Format

```
[Agent#7] feat: Implement Gateway struct with VM integration
[Agent#7] feat: Add SessionManager with trust scoring
[Agent#7] feat: Implement ProofGenerator for Proof-Carrying Actions
[Agent#7] feat: Add Router with content-addressed routing
[Agent#7] feat: Implement WebSocket server
[Agent#7] feat: Create main router graph
```

## Success Criteria

1. Gateway processes messages end-to-end
2. Sessions persist state correctly
3. Proof-Carrying Actions generate valid signatures
4. Router correctly selects skills
5. WebSocket server accepts connections
6. All operations produce execution traces

## Dependencies on Other Agents

- **Agent #0**: Need repository structure and core types
- **Agent #6**: Need execution trace support in 0-lang VM
- **Agent #6**: Need PermissionNode and RouteNode

## Handoff Points

When you complete:
- **Gateway struct**: Notify Agent #8 (can integrate channels)
- **Session Manager**: Notify Agent #9 (skills need session context)
- **Proof Generator**: Integration tests can begin
- **WebSocket Server**: Notify Agent #10 (CLI can connect)

## Questions?

Document blockers in `BLOCKERS.md` in your branch.

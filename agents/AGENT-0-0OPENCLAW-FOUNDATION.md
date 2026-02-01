# Agent #0: 0-openclaw Foundation & Coordination

## Your Mission

You are responsible for creating the foundational structure for 0-openclaw - the 0-lang rewrite of OpenClaw. You prepare everything other agents need to start their work.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-openclaw`

## What You Create

| Deliverable | Description | Blocks |
|-------------|-------------|--------|
| Repository structure | Complete folder layout | All agents |
| README.md | Value proposition - why 0-lang is better | - |
| ARCHITECTURE.md | Technical deep dive | All agents |
| Cargo.toml | Dependencies and workspace | All agents |
| Schema definitions | Cap'n Proto schemas | Agent #2, #3, #4 |
| Core traits | Base traits all agents implement | Agent #2, #3, #4 |

## Repository Structure to Create

```
0-openclaw/
├── README.md                    # YOU CREATE THIS
├── ARCHITECTURE.md              # YOU CREATE THIS
├── Cargo.toml                   # YOU CREATE THIS
├── src/
│   ├── lib.rs                   # YOU CREATE THIS (exports)
│   ├── types.rs                 # YOU CREATE THIS (core types)
│   ├── error.rs                 # YOU CREATE THIS (error types)
│   ├── gateway/                 # Agent #2 fills this
│   │   └── mod.rs              
│   ├── channels/                # Agent #3 fills this
│   │   └── mod.rs              
│   ├── skills/                  # Agent #4 fills this
│   │   └── mod.rs              
│   └── cli/                     # Agent #5 fills this
│       └── mod.rs              
├── graphs/
│   ├── core/                    # Agent #2 creates these
│   ├── channels/                # Agent #3 creates these
│   └── skills/                  # Agent #4 creates these
├── schema/
│   └── openclaw.capnp           # YOU CREATE THIS
└── examples/
    └── README.md                # YOU CREATE THIS
```

## README.md Content

The README must powerfully explain WHY 0-lang is the right choice for AI assistants.

### Required Sections

#### 1. Hero Section

```markdown
# 0-openclaw

> **Every action carries proof. Every decision is verifiable.**

The first AI assistant where you don't have to trust the code—you verify it.

[![Built with 0-lang](https://img.shields.io/badge/Built_with-0--lang-black.svg)](#)
[![Proof-Carrying](https://img.shields.io/badge/Actions-Proof--Carrying-blue.svg)](#)
```

#### 2. The Problem

```markdown
## The Problem with Traditional AI Assistants

Traditional assistants like OpenClaw, while powerful, share a fundamental flaw:

| Issue | Reality |
|-------|---------|
| **Trust-based execution** | You hope the code does what it claims |
| **Opaque decisions** | Why did it send that message? |
| **Debug by prayer** | Logs are post-hoc, incomplete |
| **Security theater** | Allowlists are just strings |
```

#### 3. The Solution

```markdown
## 0-openclaw: Proof-Carrying AI Assistant

0-openclaw rebuilds the assistant paradigm from first principles:

| Traditional | 0-openclaw |
|-------------|------------|
| Trust the code | **Verify the proof** |
| Boolean permissions | **Confidence-scored trust** |
| Text-based routing | **Content-addressed graphs** |
| Hope it's safe | **Cryptographic guarantees** |
```

#### 4. How It Works

```markdown
## How It Works

Every message through 0-openclaw produces a **Proof-Carrying Action**:

┌─────────────────────────────────────────────────────────────┐
│  Incoming Message                                           │
│       ↓                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ Parse Graph │ →  │ Route Graph │ →  │ Skill Graph │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│       ↓                   ↓                   ↓             │
│  [input_hash]      [route_trace]      [execution_trace]    │
│       └───────────────────┼───────────────────┘            │
│                           ↓                                 │
│              ┌─────────────────────────┐                   │
│              │   Proof-Carrying Action │                   │
│              │   - action: SendMessage │                   │
│              │   - proof: [hashes...]  │                   │
│              │   - confidence: 0.95    │                   │
│              │   - signature: ed25519  │                   │
│              └─────────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

#### 5. Code Comparison

Show the difference between OpenClaw TypeScript and 0-openclaw:

```markdown
## Why 0-lang? A Real Comparison

### OpenClaw (TypeScript) - Trust Required

```typescript
// You must trust this code does what it says
async function handleMessage(msg: Message) {
  if (this.allowList.includes(msg.sender)) {  // String comparison
    const response = await this.agent.process(msg);  // Black box
    await this.channel.send(response);  // Hope it's right
  }
}
```

### 0-openclaw (0-lang) - Verify Instead

```
Graph {
  name: "message_handler",
  nodes: [
    // Permission check with confidence score
    { id: 0xABC..., op: PermissionCheck, confidence_threshold: 0.8 },
    // Route based on content hash (deterministic)
    { id: 0xDEF..., op: Route, branches: [...] },
    // Execute skill with proof generation
    { id: 0x123..., op: SkillExecute, proof_required: true },
  ],
  // Every execution produces verifiable trace
  proof_policy: ProofPolicy::Always,
}
```

**The difference**: You can verify the 0-openclaw graph produces the exact behavior claimed. You cannot verify the TypeScript code without trusting the entire runtime.
```

## ARCHITECTURE.md Content

### Required Sections

#### 1. System Overview

```markdown
# 0-openclaw Architecture

## System Overview

0-openclaw is built on three pillars:

1. **Graph-Native Processing** - All logic as verifiable DAGs
2. **Proof-Carrying Actions** - Every action is cryptographically traceable
3. **Confidence-Scored Permissions** - Probabilistic trust, not boolean gates
```

#### 2. Component Diagram

```markdown
## Architecture Diagram

┌─────────────────────────────────────────────────────────────────┐
│                         0-OPENCLAW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      0-GATEWAY                            │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐          │   │
│  │  │  Session   │  │   Router   │  │   Skill    │          │   │
│  │  │  Manager   │  │   Graph    │  │  Registry  │          │   │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘          │   │
│  │        └───────────────┼───────────────┘                  │   │
│  │                        ↓                                  │   │
│  │              ┌──────────────────┐                         │   │
│  │              │      0-VM        │                         │   │
│  │              │  (Graph Engine)  │                         │   │
│  │              └────────┬─────────┘                         │   │
│  │                       ↓                                   │   │
│  │          Proof-Carrying Actions                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│         ↑              ↑              ↑              ↑          │
│    ┌────┴────┐    ┌────┴────┐    ┌────┴────┐    ┌────┴────┐    │
│    │Telegram │    │ Discord │    │  Slack  │    │   ...   │    │
│    │Connector│    │Connector│    │Connector│    │         │    │
│    └─────────┘    └─────────┘    └─────────┘    └─────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 3. Core Concepts

Document these key concepts:
- **Graph-Based Logic**: How messages flow through graphs
- **Content-Addressed Routing**: Hash-based deterministic behavior
- **Proof-Carrying Actions**: Structure and verification
- **Confidence Scores**: How permissions work probabilistically
- **Skill Composition**: How skills combine into workflows

## Cargo.toml

```toml
[package]
name = "zero-openclaw"
version = "0.1.0"
edition = "2021"
description = "Proof-carrying AI assistant built with 0-lang"
license = "Apache-2.0"
repository = "https://github.com/0-protocol/0-openclaw"

[dependencies]
# 0-lang core (local path or git)
zerolang = { path = "../0-lang" }

# Async runtime
tokio = { version = "1.0", features = ["full"] }
async-trait = "0.1"

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
capnp = "0.18"

# Cryptography
ed25519-dalek = "2.0"
sha2 = "0.10"

# HTTP/WebSocket
reqwest = { version = "0.11", features = ["json"] }
tokio-tungstenite = { version = "0.21", features = ["native-tls"] }
axum = "0.7"

# CLI
clap = { version = "4.0", features = ["derive"] }

# Logging
tracing = "0.1"
tracing-subscriber = "0.3"

# Utilities
uuid = { version = "1.0", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }

[dev-dependencies]
tokio-test = "0.4"

[build-dependencies]
capnpc = "0.18"
```

## Schema Definition (schema/openclaw.capnp)

```capnp
@0xc8d2d2f1a5e3b4c6;

# Core message types
struct Message {
  id @0 :Data;           # SHA-256 hash
  channelId @1 :Text;
  senderId @2 :Text;
  content @3 :Text;
  timestamp @4 :UInt64;
  metadata @5 :Data;     # JSON blob
}

# Proof-Carrying Action
struct ProofCarryingAction {
  action @0 :Action;
  sessionHash @1 :Data;
  inputHash @2 :Data;
  executionTrace @3 :List(Data);
  confidence @4 :Float32;
  signature @5 :Data;
  timestamp @6 :UInt64;
}

struct Action {
  union {
    sendMessage @0 :SendMessageAction;
    executeSkill @1 :ExecuteSkillAction;
    updateSession @2 :UpdateSessionAction;
  }
}

struct SendMessageAction {
  channelId @0 :Text;
  recipientId @1 :Text;
  content @2 :Text;
}

struct ExecuteSkillAction {
  skillHash @0 :Data;
  inputs @1 :Data;
}

struct UpdateSessionAction {
  sessionId @0 :Data;
  updates @1 :Data;
}

# Session state
struct Session {
  id @0 :Data;
  channelId @1 :Text;
  userId @2 :Text;
  state @3 :Data;        # Serialized state
  createdAt @4 :UInt64;
  lastActivity @5 :UInt64;
}

# Skill definition
struct Skill {
  hash @0 :Data;
  name @1 :Text;
  description @2 :Text;
  graphData @3 :Data;    # Serialized RuntimeGraph
  permissions @4 :List(Text);
}
```

## Core Types (src/types.rs)

```rust
use serde::{Deserialize, Serialize};
use sha2::{Sha256, Digest};

/// Unique identifier based on content hash
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct ContentHash(pub [u8; 32]);

impl ContentHash {
    pub fn from_bytes(data: &[u8]) -> Self {
        let mut hasher = Sha256::new();
        hasher.update(data);
        Self(hasher.finalize().into())
    }
}

/// Confidence score (0.0 to 1.0)
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct Confidence(f32);

impl Confidence {
    pub fn new(value: f32) -> Self {
        Self(value.clamp(0.0, 1.0))
    }
    
    pub fn value(&self) -> f32 {
        self.0
    }
    
    pub fn meets_threshold(&self, threshold: f32) -> bool {
        self.0 >= threshold
    }
}

/// Incoming message from any channel
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct IncomingMessage {
    pub id: ContentHash,
    pub channel_id: String,
    pub sender_id: String,
    pub content: String,
    pub timestamp: u64,
    pub metadata: serde_json::Value,
}

/// Outgoing message to any channel
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OutgoingMessage {
    pub channel_id: String,
    pub recipient_id: String,
    pub content: String,
    pub reply_to: Option<ContentHash>,
}

/// Proof-Carrying Action - the core innovation
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProofCarryingAction {
    /// The action to perform
    pub action: Action,
    
    /// Hash of the session context
    pub session_hash: ContentHash,
    
    /// Hash of the input that triggered this action
    pub input_hash: ContentHash,
    
    /// Hashes of all graph nodes evaluated
    pub execution_trace: Vec<ContentHash>,
    
    /// Confidence score for this action
    pub confidence: Confidence,
    
    /// Ed25519 signature over all fields
    pub signature: [u8; 64],
    
    /// Timestamp of action generation
    pub timestamp: u64,
}

/// Actions the assistant can take
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Action {
    SendMessage(OutgoingMessage),
    ExecuteSkill { skill_hash: ContentHash, inputs: serde_json::Value },
    UpdateSession { session_id: ContentHash, updates: serde_json::Value },
    NoOp { reason: String },
}
```

## Getting Started (for you, Agent #0)

```bash
# Create the repository
mkdir -p /Users/JiahaoRBC/Git/0-protocol/0-openclaw
cd /Users/JiahaoRBC/Git/0-protocol/0-openclaw

# Initialize git
git init
git checkout -b main

# Create directory structure
mkdir -p src/{gateway,channels,skills,cli}
mkdir -p graphs/{core,channels,skills}
mkdir -p schema
mkdir -p examples

# Create all files listed above
# Then commit
git add .
git commit -m "[Agent#0] Initialize 0-openclaw repository structure"
```

## Commit Format

```
[Agent#0] feat: Initialize repository structure
[Agent#0] docs: Create README with 0-lang value proposition
[Agent#0] docs: Create ARCHITECTURE.md
[Agent#0] feat: Add Cargo.toml with dependencies
[Agent#0] feat: Define Cap'n Proto schema
[Agent#0] feat: Implement core types
```

## Success Criteria

1. Repository structure created and committed
2. README clearly explains why 0-lang is superior
3. ARCHITECTURE.md provides clear technical guidance
4. Cargo.toml compiles successfully
5. Schema compiles with `capnp compile`
6. Core types compile and have basic tests
7. Other agents can start their work

## Handoff Points

When you complete:
- **Repository + Cargo.toml**: Notify ALL agents (they can start)
- **Schema**: Notify Agent #2, #3, #4 (they need message types)
- **Core types**: Notify Agent #2 (needs ProofCarryingAction)

## Dependencies

- Ensure `0-lang` repository exists at `../0-lang` or update Cargo.toml to use git
- Ensure Rust toolchain is installed
- Ensure Cap'n Proto compiler is installed (`brew install capnp` on macOS)

## Questions?

Document blockers in `BLOCKERS.md` in your branch.

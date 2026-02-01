# Agent #8: 0-openclaw Channel Connectors

## Your Mission

You are responsible for building channel connectors that integrate messaging platforms with the 0-openclaw Gateway. Each channel produces graph-based message processing with verifiable actions.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-openclaw`

## Channels to Implement

| Channel | Priority | Library | Features |
|---------|----------|---------|----------|
| Telegram | P0 | teloxide | Bot API, commands, groups |
| Discord | P0 | serenity | Slash commands, DMs, guilds |
| Slack | P1 | slack-morphism | Events API, commands |
| WhatsApp | P2 | TBD | Business API |

## Architecture

### Channel Trait

**File:** `src/channels/mod.rs`

```rust
use async_trait::async_trait;
use zerolang::RuntimeGraph;
use crate::types::{
    ContentHash, IncomingMessage, OutgoingMessage, 
    ProofCarryingAction, Confidence
};

/// Trait all channel connectors must implement
#[async_trait]
pub trait Channel: Send + Sync {
    /// Channel identifier
    fn name(&self) -> &str;
    
    /// Channel-specific message processing graph
    fn processing_graph(&self) -> &RuntimeGraph;
    
    /// Receive next message (blocking)
    async fn receive(&self) -> Result<IncomingMessage, ChannelError>;
    
    /// Send message and return proof
    async fn send(&self, message: OutgoingMessage) -> Result<ProofCarryingAction, ChannelError>;
    
    /// Evaluate permission for action on this channel
    fn evaluate_permission(&self, action: &Action, sender: &str) -> Confidence;
    
    /// Channel-specific allowlist
    fn allowlist(&self) -> &[String];
    
    /// Check if channel supports feature
    fn supports(&self, feature: ChannelFeature) -> bool;
}

pub enum ChannelFeature {
    Commands,      // Slash commands
    Groups,        // Group chats
    Reactions,     // Message reactions
    Threads,       // Threaded replies
    Files,         // File attachments
    Voice,         // Voice messages
}

#[derive(Debug)]
pub enum ChannelError {
    ConnectionFailed(String),
    SendFailed(String),
    ReceiveFailed(String),
    PermissionDenied(String),
    RateLimited { retry_after: u64 },
    InvalidMessage(String),
}
```

## File Structure

```
src/channels/
├── mod.rs              # Channel trait, re-exports
├── telegram/
│   ├── mod.rs          # TelegramChannel implementation
│   ├── bot.rs          # Bot setup and handlers
│   ├── commands.rs     # Command parsing
│   └── types.rs        # Telegram-specific types
├── discord/
│   ├── mod.rs          # DiscordChannel implementation
│   ├── bot.rs          # Bot setup
│   ├── commands.rs     # Slash command handling
│   └── types.rs        # Discord-specific types
├── slack/
│   ├── mod.rs          # SlackChannel implementation
│   ├── events.rs       # Events API handling
│   └── types.rs        # Slack-specific types
└── common/
    ├── mod.rs          # Shared utilities
    ├── rate_limit.rs   # Rate limiting
    └── retry.rs        # Retry logic

graphs/channels/
├── telegram.0          # Telegram processing graph
├── discord.0           # Discord processing graph
└── slack.0             # Slack processing graph
```

## Telegram Implementation

### TelegramChannel

**File:** `src/channels/telegram/mod.rs`

```rust
use teloxide::prelude::*;
use tokio::sync::mpsc;
use zerolang::RuntimeGraph;
use crate::channels::{Channel, ChannelError, ChannelFeature};
use crate::types::{ContentHash, IncomingMessage, OutgoingMessage, Confidence};

pub struct TelegramChannel {
    bot: Bot,
    processing_graph: RuntimeGraph,
    message_rx: mpsc::Receiver<IncomingMessage>,
    message_tx: mpsc::Sender<IncomingMessage>,
    config: TelegramConfig,
}

pub struct TelegramConfig {
    pub token: String,
    pub allowlist: Vec<String>,
    pub dm_policy: DmPolicy,
    pub group_policy: GroupPolicy,
}

pub enum DmPolicy {
    Open,           // Accept all DMs
    Pairing,        // Require pairing code
    Allowlist,      // Only allowlisted users
}

pub enum GroupPolicy {
    MentionOnly,    // Respond only when mentioned
    Always,         // Respond to all messages
    Disabled,       // Ignore group messages
}

impl TelegramChannel {
    pub async fn new(config: TelegramConfig) -> Result<Self, ChannelError> {
        let bot = Bot::new(&config.token);
        let (tx, rx) = mpsc::channel(100);
        
        // Load channel-specific processing graph
        let processing_graph = RuntimeGraph::load_from_file("graphs/channels/telegram.0")
            .map_err(|e| ChannelError::ConnectionFailed(e.to_string()))?;
        
        let channel = Self {
            bot,
            processing_graph,
            message_rx: rx,
            message_tx: tx,
            config,
        };
        
        // Start message listener
        channel.start_listener().await?;
        
        Ok(channel)
    }
    
    async fn start_listener(&self) -> Result<(), ChannelError> {
        let tx = self.message_tx.clone();
        let bot = self.bot.clone();
        let config = self.config.clone();
        
        tokio::spawn(async move {
            teloxide::repl(bot, move |bot: Bot, msg: Message| {
                let tx = tx.clone();
                let config = config.clone();
                
                async move {
                    // Check permissions
                    if !Self::check_permission(&msg, &config) {
                        return Ok(());
                    }
                    
                    // Convert to IncomingMessage
                    let incoming = Self::convert_message(&msg);
                    
                    // Send to channel
                    if tx.send(incoming).await.is_err() {
                        tracing::error!("Failed to send message to channel");
                    }
                    
                    Ok(())
                }
            }).await;
        });
        
        Ok(())
    }
    
    fn check_permission(msg: &Message, config: &TelegramConfig) -> bool {
        let sender_id = msg.from()
            .map(|u| u.id.to_string())
            .unwrap_or_default();
        
        // Check DM policy
        if msg.chat.is_private() {
            match config.dm_policy {
                DmPolicy::Open => true,
                DmPolicy::Allowlist => config.allowlist.contains(&sender_id),
                DmPolicy::Pairing => {
                    // Check if user has paired
                    // Implementation depends on pairing system
                    true
                }
            }
        } else {
            // Group message
            match config.group_policy {
                GroupPolicy::Disabled => false,
                GroupPolicy::MentionOnly => {
                    // Check if bot was mentioned
                    msg.text()
                        .map(|t| t.contains("@bot_username"))
                        .unwrap_or(false)
                }
                GroupPolicy::Always => true,
            }
        }
    }
    
    fn convert_message(msg: &Message) -> IncomingMessage {
        let content = msg.text()
            .or(msg.caption())
            .unwrap_or("")
            .to_string();
        
        IncomingMessage {
            id: ContentHash::from_bytes(
                format!("telegram:{}", msg.id.0).as_bytes()
            ),
            channel_id: "telegram".to_string(),
            sender_id: msg.from()
                .map(|u| u.id.to_string())
                .unwrap_or_default(),
            content,
            timestamp: msg.date.timestamp_millis() as u64,
            metadata: serde_json::json!({
                "chat_id": msg.chat.id.0,
                "message_id": msg.id.0,
                "chat_type": if msg.chat.is_private() { "private" } else { "group" },
            }),
        }
    }
}

#[async_trait]
impl Channel for TelegramChannel {
    fn name(&self) -> &str {
        "telegram"
    }
    
    fn processing_graph(&self) -> &RuntimeGraph {
        &self.processing_graph
    }
    
    async fn receive(&self) -> Result<IncomingMessage, ChannelError> {
        self.message_rx.recv().await
            .ok_or(ChannelError::ReceiveFailed("Channel closed".to_string()))
    }
    
    async fn send(&self, message: OutgoingMessage) -> Result<ProofCarryingAction, ChannelError> {
        // Parse chat_id from recipient
        let chat_id: i64 = message.recipient_id.parse()
            .map_err(|e| ChannelError::InvalidMessage(format!("Invalid chat_id: {}", e)))?;
        
        // Send message
        self.bot.send_message(ChatId(chat_id), &message.content)
            .await
            .map_err(|e| ChannelError::SendFailed(e.to_string()))?;
        
        // Note: ProofCarryingAction is generated by Gateway, not here
        // This should return success indication, Gateway wraps in PCA
        Ok(ProofCarryingAction::pending())
    }
    
    fn evaluate_permission(&self, action: &Action, sender: &str) -> Confidence {
        if self.config.allowlist.contains(&sender.to_string()) {
            Confidence::new(0.95)
        } else {
            Confidence::new(0.3)
        }
    }
    
    fn allowlist(&self) -> &[String] {
        &self.config.allowlist
    }
    
    fn supports(&self, feature: ChannelFeature) -> bool {
        match feature {
            ChannelFeature::Commands => true,
            ChannelFeature::Groups => true,
            ChannelFeature::Reactions => true,
            ChannelFeature::Threads => true,
            ChannelFeature::Files => true,
            ChannelFeature::Voice => true,
        }
    }
}
```

## Discord Implementation

**File:** `src/channels/discord/mod.rs`

```rust
use serenity::prelude::*;
use serenity::model::prelude::*;
use tokio::sync::mpsc;
use zerolang::RuntimeGraph;
use crate::channels::{Channel, ChannelError, ChannelFeature};
use crate::types::{ContentHash, IncomingMessage, OutgoingMessage, Confidence};

pub struct DiscordChannel {
    client: Client,
    processing_graph: RuntimeGraph,
    message_rx: mpsc::Receiver<IncomingMessage>,
    config: DiscordConfig,
}

pub struct DiscordConfig {
    pub token: String,
    pub application_id: u64,
    pub dm_allowlist: Vec<String>,
    pub guild_allowlist: Vec<u64>,
    pub register_commands: bool,
}

struct Handler {
    tx: mpsc::Sender<IncomingMessage>,
}

#[async_trait]
impl EventHandler for Handler {
    async fn message(&self, _ctx: Context, msg: serenity::model::channel::Message) {
        // Skip bot messages
        if msg.author.bot {
            return;
        }
        
        let incoming = IncomingMessage {
            id: ContentHash::from_bytes(
                format!("discord:{}", msg.id.0).as_bytes()
            ),
            channel_id: "discord".to_string(),
            sender_id: msg.author.id.to_string(),
            content: msg.content.clone(),
            timestamp: msg.timestamp.timestamp_millis() as u64,
            metadata: serde_json::json!({
                "channel_id": msg.channel_id.to_string(),
                "guild_id": msg.guild_id.map(|g| g.to_string()),
                "message_id": msg.id.to_string(),
            }),
        };
        
        let _ = self.tx.send(incoming).await;
    }
    
    async fn interaction_create(&self, ctx: Context, interaction: Interaction) {
        if let Interaction::Command(command) = interaction {
            // Handle slash commands
            let content = format!(
                "/{} {}", 
                command.data.name,
                command.data.options.iter()
                    .map(|o| format!("{}={:?}", o.name, o.value))
                    .collect::<Vec<_>>()
                    .join(" ")
            );
            
            let incoming = IncomingMessage {
                id: ContentHash::from_bytes(
                    format!("discord:cmd:{}", command.id.0).as_bytes()
                ),
                channel_id: "discord".to_string(),
                sender_id: command.user.id.to_string(),
                content,
                timestamp: chrono::Utc::now().timestamp_millis() as u64,
                metadata: serde_json::json!({
                    "type": "slash_command",
                    "command": command.data.name,
                    "interaction_id": command.id.to_string(),
                }),
            };
            
            let _ = self.tx.send(incoming).await;
        }
    }
    
    async fn ready(&self, ctx: Context, ready: Ready) {
        tracing::info!("Discord bot ready as {}", ready.user.name);
    }
}

impl DiscordChannel {
    pub async fn new(config: DiscordConfig) -> Result<Self, ChannelError> {
        let (tx, rx) = mpsc::channel(100);
        
        let intents = GatewayIntents::GUILD_MESSAGES
            | GatewayIntents::DIRECT_MESSAGES
            | GatewayIntents::MESSAGE_CONTENT;
        
        let client = Client::builder(&config.token, intents)
            .event_handler(Handler { tx })
            .await
            .map_err(|e| ChannelError::ConnectionFailed(e.to_string()))?;
        
        let processing_graph = RuntimeGraph::load_from_file("graphs/channels/discord.0")
            .map_err(|e| ChannelError::ConnectionFailed(e.to_string()))?;
        
        Ok(Self {
            client,
            processing_graph,
            message_rx: rx,
            config,
        })
    }
}

#[async_trait]
impl Channel for DiscordChannel {
    fn name(&self) -> &str {
        "discord"
    }
    
    fn processing_graph(&self) -> &RuntimeGraph {
        &self.processing_graph
    }
    
    async fn receive(&self) -> Result<IncomingMessage, ChannelError> {
        self.message_rx.recv().await
            .ok_or(ChannelError::ReceiveFailed("Channel closed".to_string()))
    }
    
    async fn send(&self, message: OutgoingMessage) -> Result<ProofCarryingAction, ChannelError> {
        // Implementation
        Ok(ProofCarryingAction::pending())
    }
    
    fn evaluate_permission(&self, action: &Action, sender: &str) -> Confidence {
        if self.config.dm_allowlist.contains(&sender.to_string()) {
            Confidence::new(0.95)
        } else {
            Confidence::new(0.3)
        }
    }
    
    fn allowlist(&self) -> &[String] {
        &self.config.dm_allowlist
    }
    
    fn supports(&self, feature: ChannelFeature) -> bool {
        match feature {
            ChannelFeature::Commands => true,
            ChannelFeature::Groups => true,
            ChannelFeature::Reactions => true,
            ChannelFeature::Threads => true,
            ChannelFeature::Files => true,
            ChannelFeature::Voice => false, // Not implemented
        }
    }
}
```

## Slack Implementation

**File:** `src/channels/slack/mod.rs`

```rust
use slack_morphism::prelude::*;
use crate::channels::{Channel, ChannelError, ChannelFeature};
use crate::types::{ContentHash, IncomingMessage, OutgoingMessage, Confidence};

pub struct SlackChannel {
    client: SlackClient,
    processing_graph: RuntimeGraph,
    message_rx: mpsc::Receiver<IncomingMessage>,
    config: SlackConfig,
}

pub struct SlackConfig {
    pub bot_token: String,
    pub app_token: String,
    pub workspace_allowlist: Vec<String>,
    pub channel_allowlist: Vec<String>,
}

impl SlackChannel {
    pub async fn new(config: SlackConfig) -> Result<Self, ChannelError> {
        // Implementation similar to Telegram/Discord
        unimplemented!()
    }
}

#[async_trait]
impl Channel for SlackChannel {
    // Similar implementation pattern
}
```

## Channel Processing Graphs

### graphs/channels/telegram.0

```
# Telegram Message Processing Graph
#
# Processes Telegram-specific message features before routing.

Graph {
    name: "telegram_processor",
    version: 1,
    
    nodes: [
        { id: "input", type: External, uri: "input://message" },
        
        # Extract command if present
        { id: "is_command", type: Operation, op: StartsWith, inputs: ["input", "/"] },
        { id: "command", type: Branch, 
          condition: "is_command",
          true_branch: "extract_command",
          false_branch: "pass_through" 
        },
        
        # Handle /start command specially
        { id: "is_start", type: Operation, op: Eq, inputs: ["command", "/start"] },
        
        # Permission check
        { id: "permission", type: Permission,
          subject: "sender",
          action: "command",
          threshold: 0.7,
          fallback: "deny_response"
        },
        
        { id: "output", type: Operation, op: Identity, inputs: ["permission"] },
    ],
    
    outputs: ["output"],
}
```

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-openclaw
git checkout -b agent8/channels

# Wait for Agent #0 and Agent #7, then:
cat src/lib.rs
cat src/gateway/mod.rs

# Create channel modules
mkdir -p src/channels/{telegram,discord,slack,common}
touch src/channels/mod.rs
touch src/channels/telegram/mod.rs
touch src/channels/discord/mod.rs
touch src/channels/slack/mod.rs
```

## Dependencies to Add (Cargo.toml)

```toml
# Telegram
teloxide = { version = "0.12", features = ["auto-send", "webhooks"] }

# Discord
serenity = { version = "0.12", features = ["client", "gateway", "model"] }

# Slack
slack-morphism = "1.0"
```

## Commit Format

```
[Agent#8] feat: Define Channel trait with graph support
[Agent#8] feat: Implement TelegramChannel
[Agent#8] feat: Implement DiscordChannel with slash commands
[Agent#8] feat: Implement SlackChannel
[Agent#8] feat: Add channel processing graphs
[Agent#8] feat: Add rate limiting and retry logic
```

## Success Criteria

1. Telegram bot connects and receives messages
2. Discord bot handles slash commands
3. Slack bot processes events
4. All channels produce proper IncomingMessage format
5. Permission evaluation works correctly
6. Processing graphs execute successfully

## Dependencies on Other Agents

- **Agent #0**: Need repository structure and types
- **Agent #6**: Need StreamTensor for message streams
- **Agent #7**: Need Gateway to register channels with

## Handoff Points

When you complete:
- **Channel trait**: Notify Agent #9 (skills may use channel info)
- **TelegramChannel**: Integration tests can begin
- **All channels**: Notify Agent #10 (CLI can start all channels)

## Questions?

Document blockers in `BLOCKERS.md` in your branch.

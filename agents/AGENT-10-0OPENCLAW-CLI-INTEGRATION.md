# Agent #10: 0-openclaw CLI & Integration

## Your Mission

You are the **final integration agent** responsible for:
1. Building the CLI interface
2. Integrating all components from other agents
3. Final testing and documentation
4. Merging all branches into main

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-openclaw`

## Components to Build

| Component | Priority | Description |
|-----------|----------|-------------|
| CLI | P0 | Command-line interface |
| Configuration | P0 | Config file management |
| Integration Tests | P0 | End-to-end testing |
| Docker Setup | P1 | Container deployment |
| Documentation | P1 | User and developer docs |
| Branch Merging | P0 | Final integration |

## File Structure

```
src/cli/
├── mod.rs              # CLI main entry
├── commands/
│   ├── mod.rs          # Command definitions
│   ├── gateway.rs      # Gateway commands
│   ├── channel.rs      # Channel commands
│   ├── skill.rs        # Skill commands
│   └── config.rs       # Config commands
└── config.rs           # Configuration management

tests/
├── integration/
│   ├── mod.rs
│   ├── gateway_test.rs
│   ├── channel_test.rs
│   └── skill_test.rs
└── e2e/
    ├── mod.rs
    └── full_flow_test.rs

docs/
├── getting-started.md
├── configuration.md
├── channels/
│   ├── telegram.md
│   ├── discord.md
│   └── slack.md
├── skills/
│   ├── overview.md
│   ├── creating-skills.md
│   └── composition.md
└── api/
    └── gateway-api.md

docker/
├── Dockerfile
├── docker-compose.yml
└── .env.example
```

## CLI Implementation

### Main Entry Point

**File:** `src/main.rs`

```rust
use clap::{Parser, Subcommand};
use zero_openclaw::{Gateway, GatewayConfig};
use std::path::PathBuf;

#[derive(Parser)]
#[command(name = "0-openclaw")]
#[command(about = "Proof-carrying AI assistant built with 0-lang")]
#[command(version)]
struct Cli {
    /// Config file path
    #[arg(short, long, default_value = "~/.0-openclaw/config.json")]
    config: PathBuf,
    
    /// Verbose output
    #[arg(short, long)]
    verbose: bool,
    
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Start the gateway
    Gateway {
        /// Port to listen on
        #[arg(short, long, default_value = "18789")]
        port: u16,
        
        /// Run in daemon mode
        #[arg(short, long)]
        daemon: bool,
    },
    
    /// Channel management
    Channel {
        #[command(subcommand)]
        action: ChannelCommands,
    },
    
    /// Skill management
    Skill {
        #[command(subcommand)]
        action: SkillCommands,
    },
    
    /// Configuration management
    Config {
        #[command(subcommand)]
        action: ConfigCommands,
    },
    
    /// Send a test message
    Message {
        /// Channel to send to
        #[arg(short, long)]
        channel: String,
        
        /// Recipient ID
        #[arg(short, long)]
        to: String,
        
        /// Message content
        #[arg(short, long)]
        message: String,
    },
    
    /// Verify a proof-carrying action
    Verify {
        /// Path to PCA file
        pca_file: PathBuf,
    },
    
    /// Show gateway status
    Status,
    
    /// Run diagnostics
    Doctor,
    
    /// Initialize a new 0-openclaw installation
    Init {
        /// Installation directory
        #[arg(default_value = "~/.0-openclaw")]
        path: PathBuf,
    },
}

#[derive(Subcommand)]
enum ChannelCommands {
    /// List connected channels
    List,
    
    /// Connect a channel
    Connect {
        /// Channel type (telegram, discord, slack)
        channel_type: String,
    },
    
    /// Disconnect a channel
    Disconnect {
        /// Channel name
        name: String,
    },
    
    /// Show channel status
    Status {
        /// Channel name
        name: String,
    },
    
    /// Login to a channel
    Login {
        /// Channel type
        channel_type: String,
    },
}

#[derive(Subcommand)]
enum SkillCommands {
    /// List installed skills
    List,
    
    /// Install a skill
    Install {
        /// Skill path or URL
        source: String,
    },
    
    /// Uninstall a skill
    Uninstall {
        /// Skill name or hash
        skill: String,
    },
    
    /// Verify a skill
    Verify {
        /// Skill name or hash
        skill: String,
    },
    
    /// Show skill info
    Info {
        /// Skill name or hash
        skill: String,
    },
    
    /// Compose multiple skills
    Compose {
        /// Skill names to compose
        skills: Vec<String>,
        
        /// Output name
        #[arg(short, long)]
        output: String,
    },
}

#[derive(Subcommand)]
enum ConfigCommands {
    /// Show current config
    Show,
    
    /// Set a config value
    Set {
        /// Config key
        key: String,
        
        /// Config value
        value: String,
    },
    
    /// Get a config value
    Get {
        /// Config key
        key: String,
    },
    
    /// Validate config
    Validate,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cli = Cli::parse();
    
    // Initialize logging
    let log_level = if cli.verbose { "debug" } else { "info" };
    tracing_subscriber::fmt()
        .with_env_filter(log_level)
        .init();
    
    // Load config
    let config = load_config(&cli.config)?;
    
    match cli.command {
        Commands::Gateway { port, daemon } => {
            run_gateway(config, port, daemon).await?;
        }
        Commands::Channel { action } => {
            handle_channel_command(action, &config).await?;
        }
        Commands::Skill { action } => {
            handle_skill_command(action, &config).await?;
        }
        Commands::Config { action } => {
            handle_config_command(action, &cli.config)?;
        }
        Commands::Message { channel, to, message } => {
            send_message(&config, &channel, &to, &message).await?;
        }
        Commands::Verify { pca_file } => {
            verify_pca(&pca_file)?;
        }
        Commands::Status => {
            show_status(&config).await?;
        }
        Commands::Doctor => {
            run_doctor(&config).await?;
        }
        Commands::Init { path } => {
            init_installation(&path)?;
        }
    }
    
    Ok(())
}

async fn run_gateway(
    config: Config, 
    port: u16, 
    daemon: bool
) -> Result<(), Box<dyn std::error::Error>> {
    println!("Starting 0-openclaw gateway on port {}...", port);
    
    let gateway_config = GatewayConfig::from_config(&config)?;
    let gateway = Gateway::new(gateway_config)?;
    
    // Register channels
    for channel_config in &config.channels {
        match channel_config.channel_type.as_str() {
            "telegram" => {
                let channel = TelegramChannel::new(channel_config.into()).await?;
                gateway.register_channel(Arc::new(channel));
            }
            "discord" => {
                let channel = DiscordChannel::new(channel_config.into()).await?;
                gateway.register_channel(Arc::new(channel));
            }
            "slack" => {
                let channel = SlackChannel::new(channel_config.into()).await?;
                gateway.register_channel(Arc::new(channel));
            }
            _ => {
                tracing::warn!("Unknown channel type: {}", channel_config.channel_type);
            }
        }
    }
    
    // Load skills
    gateway.skills.load_builtin().await?;
    for skill_path in &config.skills {
        gateway.skills.install_from_file(skill_path)?;
    }
    
    if daemon {
        // Daemonize
        daemonize::Daemonize::new()
            .pid_file("/tmp/0-openclaw.pid")
            .start()?;
    }
    
    // Start gateway
    gateway.run().await?;
    
    Ok(())
}

async fn show_status(config: &Config) -> Result<(), Box<dyn std::error::Error>> {
    println!("═══════════════════════════════════════════════════");
    println!("              0-OPENCLAW STATUS                     ");
    println!("═══════════════════════════════════════════════════");
    
    // Check gateway connection
    let gateway_status = check_gateway_status(config).await;
    println!("Gateway:     {}", if gateway_status { "✓ Running" } else { "✗ Not running" });
    
    // Check channels
    println!("\nChannels:");
    for channel in &config.channels {
        let status = check_channel_status(&channel.channel_type).await;
        println!("  {}: {}", channel.channel_type, 
                 if status { "✓ Connected" } else { "✗ Disconnected" });
    }
    
    // Check skills
    println!("\nSkills:");
    let skill_count = count_installed_skills(config)?;
    println!("  Installed: {} skills", skill_count);
    
    println!("═══════════════════════════════════════════════════");
    
    Ok(())
}

async fn run_doctor(config: &Config) -> Result<(), Box<dyn std::error::Error>> {
    println!("Running 0-openclaw diagnostics...\n");
    
    let mut issues = Vec::new();
    
    // Check config validity
    print!("Checking configuration... ");
    if let Err(e) = validate_config(config) {
        println!("✗");
        issues.push(format!("Config error: {}", e));
    } else {
        println!("✓");
    }
    
    // Check 0-lang installation
    print!("Checking 0-lang... ");
    if check_zerolang_installed() {
        println!("✓");
    } else {
        println!("✗");
        issues.push("0-lang not found".to_string());
    }
    
    // Check channel credentials
    print!("Checking channel credentials... ");
    let cred_issues = check_channel_credentials(config);
    if cred_issues.is_empty() {
        println!("✓");
    } else {
        println!("✗");
        issues.extend(cred_issues);
    }
    
    // Check skill graphs
    print!("Verifying skill graphs... ");
    let skill_issues = verify_all_skills(config)?;
    if skill_issues.is_empty() {
        println!("✓");
    } else {
        println!("✗");
        issues.extend(skill_issues);
    }
    
    // Summary
    println!("\n═══════════════════════════════════════════════════");
    if issues.is_empty() {
        println!("All checks passed! ✓");
    } else {
        println!("Found {} issue(s):", issues.len());
        for issue in &issues {
            println!("  • {}", issue);
        }
    }
    println!("═══════════════════════════════════════════════════");
    
    Ok(())
}
```

### Configuration Management

**File:** `src/cli/config.rs`

```rust
use serde::{Deserialize, Serialize};
use std::path::Path;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Config {
    /// Gateway settings
    pub gateway: GatewaySettings,
    
    /// Channel configurations
    pub channels: Vec<ChannelConfig>,
    
    /// Skill paths to load
    pub skills: Vec<String>,
    
    /// Agent settings
    pub agent: AgentSettings,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GatewaySettings {
    pub port: u16,
    pub bind: String,
    pub keypair_path: String,
    pub router_graph: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChannelConfig {
    pub channel_type: String,
    pub enabled: bool,
    pub token: Option<String>,
    pub allowlist: Vec<String>,
    #[serde(flatten)]
    pub extra: serde_json::Value,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentSettings {
    pub model: String,
    pub workspace: String,
}

impl Config {
    pub fn load(path: &Path) -> Result<Self, ConfigError> {
        let content = std::fs::read_to_string(path)?;
        let config: Config = serde_json::from_str(&content)?;
        config.validate()?;
        Ok(config)
    }
    
    pub fn save(&self, path: &Path) -> Result<(), ConfigError> {
        let content = serde_json::to_string_pretty(self)?;
        std::fs::write(path, content)?;
        Ok(())
    }
    
    pub fn validate(&self) -> Result<(), ConfigError> {
        // Validate all config values
        if self.gateway.port == 0 {
            return Err(ConfigError::InvalidValue("gateway.port".to_string()));
        }
        
        for channel in &self.channels {
            if channel.enabled && channel.token.is_none() {
                return Err(ConfigError::MissingValue(
                    format!("channels.{}.token", channel.channel_type)
                ));
            }
        }
        
        Ok(())
    }
    
    pub fn default() -> Self {
        Self {
            gateway: GatewaySettings {
                port: 18789,
                bind: "127.0.0.1".to_string(),
                keypair_path: "~/.0-openclaw/keypair".to_string(),
                router_graph: "graphs/core/router.0".to_string(),
            },
            channels: vec![],
            skills: vec![
                "graphs/skills/echo.0".to_string(),
            ],
            agent: AgentSettings {
                model: "anthropic/claude-opus-4-5".to_string(),
                workspace: "~/.0-openclaw/workspace".to_string(),
            },
        }
    }
}
```

## Integration Tests

**File:** `tests/integration/full_flow_test.rs`

```rust
use zero_openclaw::{Gateway, GatewayConfig, TelegramChannel, SkillRegistry};
use zero_openclaw::types::{IncomingMessage, ContentHash, Confidence};

#[tokio::test]
async fn test_full_message_flow() {
    // 1. Setup gateway
    let config = GatewayConfig::test_config();
    let gateway = Gateway::new(config).unwrap();
    
    // 2. Install test skill
    let skill_graph = create_test_skill();
    gateway.skills.install_graph("test_skill", skill_graph, false).unwrap();
    
    // 3. Create test message
    let message = IncomingMessage {
        id: ContentHash::from_bytes(b"test_message_1"),
        channel_id: "test".to_string(),
        sender_id: "test_user".to_string(),
        content: "/echo Hello World".to_string(),
        timestamp: chrono::Utc::now().timestamp_millis() as u64,
        metadata: serde_json::json!({}),
    };
    
    // 4. Process message
    let pca = gateway.process_message(message).await.unwrap();
    
    // 5. Verify proof-carrying action
    assert!(gateway.proof_generator.verify(&pca).unwrap());
    assert!(pca.confidence.value() > 0.8);
    assert!(!pca.execution_trace.is_empty());
    
    // 6. Check action is correct
    match &pca.action {
        Action::SendMessage(msg) => {
            assert!(msg.content.contains("Hello World"));
        }
        _ => panic!("Expected SendMessage action"),
    }
}

#[tokio::test]
async fn test_skill_composition() {
    let registry = SkillRegistry::new("test_skills");
    
    // Install two skills
    let skill_a = create_skill_a();
    let skill_b = create_skill_b();
    
    let hash_a = registry.install_graph("skill_a", skill_a, false).unwrap();
    let hash_b = registry.install_graph("skill_b", skill_b, false).unwrap();
    
    // Compose skills
    let mut composer = SkillComposer::new();
    composer.add_skill(registry.get(&hash_a).unwrap().graph.clone());
    composer.add_skill(registry.get(&hash_b).unwrap().graph.clone());
    composer.connect(hash_a, "output", hash_b, "input");
    
    let composed = composer.compose().unwrap();
    
    // Verify composed skill
    let verification = SkillVerifier::verify(&composed.graph).unwrap();
    assert!(verification.safe);
}

#[tokio::test]
async fn test_permission_confidence() {
    let config = GatewayConfig::test_config();
    let gateway = Gateway::new(config).unwrap();
    
    // Test allowlisted user
    let allowlisted_message = IncomingMessage {
        sender_id: "allowed_user".to_string(),
        // ...
    };
    let pca = gateway.process_message(allowlisted_message).await.unwrap();
    assert!(pca.confidence.value() > 0.9);
    
    // Test unknown user
    let unknown_message = IncomingMessage {
        sender_id: "unknown_user".to_string(),
        // ...
    };
    let pca = gateway.process_message(unknown_message).await.unwrap();
    assert!(pca.confidence.value() < 0.5);
}
```

## Docker Setup

**File:** `docker/Dockerfile`

```dockerfile
FROM rust:1.75-slim as builder

WORKDIR /app

# Install dependencies
RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    capnproto \
    && rm -rf /var/lib/apt/lists/*

# Copy source
COPY . .

# Build
RUN cargo build --release

# Runtime stage
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy binary
COPY --from=builder /app/target/release/zero-openclaw /usr/local/bin/

# Copy default config and graphs
COPY --from=builder /app/graphs /app/graphs

# Create data directory
RUN mkdir -p /data

EXPOSE 18789

ENTRYPOINT ["zero-openclaw"]
CMD ["gateway", "--port", "18789"]
```

**File:** `docker/docker-compose.yml`

```yaml
version: '3.8'

services:
  gateway:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    ports:
      - "18789:18789"
    volumes:
      - openclaw-data:/data
      - ./config.json:/app/config.json:ro
    environment:
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
      - DISCORD_BOT_TOKEN=${DISCORD_BOT_TOKEN}
      - SLACK_BOT_TOKEN=${SLACK_BOT_TOKEN}
    restart: unless-stopped

volumes:
  openclaw-data:
```

## Branch Merging Process

As the final integration agent, you are responsible for merging all branches:

### Merge Order

```
1. Agent #0 (foundation) → main
2. Agent #6 (0-lang extensions) → main (separate repo)
3. Agent #7 (gateway) → main
4. Agent #8 (channels) → main
5. Agent #9 (skills) → main
6. Agent #10 (CLI/integration) → main
```

### Merge Checklist

For each branch merge:

```bash
# 1. Ensure branch is up to date
git checkout agent7/gateway
git rebase main

# 2. Run tests
cargo test

# 3. Check for conflicts
git checkout main
git merge --no-commit --no-ff agent7/gateway

# 4. Resolve conflicts if any
# 5. Run full test suite
cargo test --all

# 6. Complete merge
git commit -m "Merge agent7/gateway: Gateway implementation"
```

### Final Integration Checklist

- [ ] All agent branches merged
- [ ] All tests pass (`cargo test --all`)
- [ ] Documentation complete
- [ ] Docker build works
- [ ] CLI commands work
- [ ] At least one channel connects successfully
- [ ] Skills load and execute
- [ ] Proof-carrying actions generate valid signatures

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-openclaw
git checkout -b agent10/cli-integration

# Wait for other agents to complete, then:
# Read their code
cat src/gateway/mod.rs
cat src/channels/mod.rs
cat src/skills/mod.rs

# Create CLI module
mkdir -p src/cli/commands
touch src/cli/mod.rs
touch src/cli/config.rs

# Create test directories
mkdir -p tests/{integration,e2e}

# Create docs
mkdir -p docs/{channels,skills,api}
```

## Commit Format

```
[Agent#10] feat: Implement CLI main entry point
[Agent#10] feat: Add gateway command
[Agent#10] feat: Add channel management commands
[Agent#10] feat: Add skill management commands
[Agent#10] feat: Add configuration management
[Agent#10] test: Add integration tests
[Agent#10] docs: Add getting started guide
[Agent#10] chore: Add Docker setup
[Agent#10] chore: Merge agent7/gateway into main
[Agent#10] chore: Merge agent8/channels into main
[Agent#10] chore: Merge agent9/skills into main
[Agent#10] release: v0.1.0
```

## Success Criteria

1. CLI works for all commands
2. Configuration loads and validates correctly
3. All integration tests pass
4. Docker container builds and runs
5. Documentation is complete and accurate
6. All agent branches successfully merged
7. End-to-end message flow works

## Dependencies on Other Agents

- **ALL AGENTS**: You depend on everyone
- **Agent #0**: Repository structure
- **Agent #7**: Gateway implementation
- **Agent #8**: Channel implementations
- **Agent #9**: Skills implementation

## Final Deliverable

When complete, 0-openclaw should:

1. Install via `cargo install zero-openclaw`
2. Initialize with `0-openclaw init`
3. Start gateway with `0-openclaw gateway`
4. Connect channels with `0-openclaw channel connect telegram`
5. Process messages and generate proof-carrying actions
6. Verify actions with `0-openclaw verify`

## Questions?

Document blockers in `BLOCKERS.md` in your branch.

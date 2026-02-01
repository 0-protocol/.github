# Agent #3: DEX Connector Specialist

## Your Mission

You are responsible for building decentralized exchange (DEX) connectors.

**Repository:** `/Users/JiahaoRBC/Git/0-protocol/0-hummingbot`

## Connectors to Implement

| Exchange | Type | Chain | Priority | API Docs |
|----------|------|-------|----------|----------|
| Hyperliquid | Perpetual DEX | Arbitrum | P0 | https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api |
| dYdX v4 | Perpetual DEX | Cosmos | P1 | https://docs.dydx.exchange/ |
| Jupiter | Swap Aggregator | Solana | P2 | https://station.jup.ag/docs |

## Architecture

Use the same `ConnectorBase` trait from Agent #2, extended for DEX-specific features.

### DEX Extension Trait

**File:** `src/connectors/dex/mod.rs`

```rust
use crate::connectors::ConnectorBase;

#[async_trait]
pub trait DexConnector: ConnectorBase {
    /// Get wallet address
    fn wallet_address(&self) -> &str;
    
    /// Sign a message
    async fn sign_message(&self, message: &[u8]) -> Result<Vec<u8>, ConnectorError>;
    
    /// Get on-chain gas estimate
    async fn estimate_gas(&self, action: &str) -> Result<Decimal, ConnectorError>;
    
    /// Wait for transaction confirmation
    async fn wait_for_confirmation(&self, tx_hash: &str) -> Result<TxReceipt, ConnectorError>;
}
```

## File Structure

```
src/connectors/dex/
├── mod.rs              # DexConnector trait, re-exports
├── hyperliquid/
│   ├── mod.rs
│   ├── api.rs          # REST/WS API
│   ├── signing.rs      # EIP-712 signing
│   └── types.rs        # Hyperliquid-specific types
├── dydx/
│   ├── mod.rs
│   ├── client.rs       # Cosmos gRPC client
│   └── signing.rs      # Cosmos signing
└── jupiter/
    ├── mod.rs
    ├── api.rs          # Jupiter API
    └── signing.rs      # Solana signing

src/wallet/
├── mod.rs
├── evm.rs              # EVM wallet (ethers-rs)
├── cosmos.rs           # Cosmos wallet
└── solana.rs           # Solana wallet
```

## Hyperliquid Implementation

Hyperliquid is the priority - it's the most popular on-chain perpetual DEX for HFT.

**File:** `src/connectors/dex/hyperliquid/mod.rs`

```rust
use ethers::signers::{LocalWallet, Signer};
use crate::connectors::{ConnectorBase, DexConnector, types::*};

pub struct HyperliquidConnector {
    wallet: LocalWallet,
    api_url: String,
    ws_url: String,
    client: reqwest::Client,
}

impl HyperliquidConnector {
    pub fn new(private_key: &str) -> Result<Self, ConnectorError> {
        let wallet: LocalWallet = private_key.parse()?;
        Ok(Self {
            wallet,
            api_url: "https://api.hyperliquid.xyz".to_string(),
            ws_url: "wss://api.hyperliquid.xyz/ws".to_string(),
            client: reqwest::Client::new(),
        })
    }
}

#[async_trait]
impl ConnectorBase for HyperliquidConnector {
    fn name(&self) -> &str { "hyperliquid" }
    
    async fn get_ticker(&self, pair: &str) -> Result<Ticker, ConnectorError> {
        // POST to /info with {"type": "allMids"}
        let body = serde_json::json!({
            "type": "allMids"
        });
        let resp = self.client.post(&format!("{}/info", self.api_url))
            .json(&body)
            .send()
            .await?;
        // Parse response...
    }
    
    async fn place_order(&self, order: &OrderRequest) -> Result<OrderResponse, ConnectorError> {
        // Hyperliquid requires EIP-712 signed orders
        let order_action = self.build_order_action(order);
        let signature = self.sign_l1_action(&order_action).await?;
        
        let body = serde_json::json!({
            "action": order_action,
            "nonce": chrono::Utc::now().timestamp_millis(),
            "signature": signature,
        });
        
        let resp = self.client.post(&format!("{}/exchange", self.api_url))
            .json(&body)
            .send()
            .await?;
        // Parse response...
    }
}

#[async_trait]
impl DexConnector for HyperliquidConnector {
    fn wallet_address(&self) -> &str {
        // Return checksummed address
    }
    
    async fn sign_message(&self, message: &[u8]) -> Result<Vec<u8>, ConnectorError> {
        let signature = self.wallet.sign_message(message).await?;
        Ok(signature.to_vec())
    }
}
```

### Hyperliquid Order Signing (EIP-712)

**File:** `src/connectors/dex/hyperliquid/signing.rs`

```rust
use ethers::types::{H256, U256};
use ethers::core::types::transaction::eip712::{Eip712, TypedData};

pub fn build_order_typed_data(order: &OrderAction) -> TypedData {
    // Hyperliquid uses EIP-712 typed data for signing
    // Domain: { name: "HyperliquidSignTransaction", version: "1", chainId: 42161, verifyingContract: "0x..." }
    // Types: Order { ... }
}

pub async fn sign_l1_action(wallet: &LocalWallet, action: &serde_json::Value) -> Result<String, Error> {
    let typed_data = build_typed_data(action);
    let signature = wallet.sign_typed_data(&typed_data).await?;
    Ok(format!("{:?}", signature))
}
```

## Wallet Integration

### EVM Wallet

**File:** `src/wallet/evm.rs`

```rust
use ethers::signers::{LocalWallet, Signer};
use ethers::providers::{Provider, Http};

pub struct EvmWallet {
    wallet: LocalWallet,
    provider: Provider<Http>,
}

impl EvmWallet {
    pub fn from_private_key(private_key: &str, rpc_url: &str) -> Result<Self, WalletError> {
        let wallet: LocalWallet = private_key.parse()?;
        let provider = Provider::<Http>::try_from(rpc_url)?;
        Ok(Self { wallet, provider })
    }
    
    pub fn address(&self) -> String {
        format!("{:?}", self.wallet.address())
    }
    
    pub async fn sign_message(&self, message: &[u8]) -> Result<Vec<u8>, WalletError> {
        let signature = self.wallet.sign_message(message).await?;
        Ok(signature.to_vec())
    }
}
```

### Solana Wallet

**File:** `src/wallet/solana.rs`

```rust
use solana_sdk::signature::{Keypair, Signer};
use solana_sdk::pubkey::Pubkey;

pub struct SolanaWallet {
    keypair: Keypair,
    rpc_url: String,
}

impl SolanaWallet {
    pub fn from_private_key(private_key: &[u8]) -> Result<Self, WalletError> {
        let keypair = Keypair::from_bytes(private_key)?;
        Ok(Self {
            keypair,
            rpc_url: "https://api.mainnet-beta.solana.com".to_string(),
        })
    }
    
    pub fn pubkey(&self) -> Pubkey {
        self.keypair.pubkey()
    }
    
    pub fn sign(&self, message: &[u8]) -> Vec<u8> {
        self.keypair.sign_message(message).as_ref().to_vec()
    }
}
```

## Getting Started

```bash
cd /Users/JiahaoRBC/Git/0-protocol/0-hummingbot
git checkout -b agent3/dex-connectors

# Create directory structure
mkdir -p src/connectors/dex/hyperliquid
mkdir -p src/connectors/dex/dydx
mkdir -p src/connectors/dex/jupiter
mkdir -p src/wallet
```

## Dependencies to Add (Cargo.toml)

```toml
# EVM (Hyperliquid, dYdX bridge)
ethers = { version = "2.0", features = ["legacy", "rustls"] }

# Solana (Jupiter)
solana-sdk = "1.17"
solana-client = "1.17"

# HTTP client
reqwest = { version = "0.11", features = ["json"] }

# WebSocket
tokio-tungstenite = { version = "0.20", features = ["native-tls"] }

# Async
async-trait = "0.1"
tokio = { version = "1.0", features = ["full"] }

# Data
rust_decimal = "1.33"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

## Commit Format

```
[Agent#3] feat: Add DexConnector trait
[Agent#3] feat: Implement EVM wallet
[Agent#3] feat: Implement Hyperliquid connector
[Agent#3] feat: Add Hyperliquid EIP-712 signing
[Agent#3] feat: Implement dYdX v4 connector
```

## Testing Strategy

1. Unit tests with mock responses
2. Testnet testing:
   - Hyperliquid: Use testnet at https://api.hyperliquid-testnet.xyz
   - dYdX: Use testnet
   - Jupiter: Use devnet

## Priority Order

1. **Hyperliquid** - Most important for HFT, simpler API
2. **EVM wallet** - Needed for Hyperliquid signing
3. **dYdX v4** - Cosmos-based, more complex
4. **Jupiter** - Solana swaps, different paradigm
5. **Solana wallet** - Needed for Jupiter

## Dependencies on Other Agents

- **Agent #1**: Need async execution for on-chain confirmations
- **Agent #2**: Use same ConnectorBase trait interface

## Handoff Points

When you complete:
- **DexConnector trait**: Share interface with Agent #5
- **Hyperliquid connector**: Notify Agent #4 (can test DEX-CEX arbitrage)
- **All DEX connectors**: Agent #5 can build full CLI

## Questions?

Document blockers in `BLOCKERS.md` in your branch.

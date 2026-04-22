# Cargo Setup & Migration

This guide covers dependency management, feature flags, crate selection patterns, and migrating from ethers-rs to Alloy.

## Quick Dependency Patterns

### Simple Script (Meta-crate)

For small scripts and quick prototypes where you want everything in one dependency:

```toml
[dependencies]
alloy = { version = "2.0", features = ["full"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
eyre = "0.6"
```

### Provider + Signer (Most Common)

For applications that send transactions:

```toml
[dependencies]
alloy-primitives = "2.0"
alloy-provider = { version = "2.0", features = ["reqwest"] }
alloy-signer-local = "2.0"
alloy-sol-types = "2.0"
alloy-network = { version = "2.0", features = ["ethereum"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
eyre = "0.6"
```

### Contract Interaction Only (No Signing)

For read-only applications (blockchain explorers, indexers, dashboards):

```toml
[dependencies]
alloy-primitives = "2.0"
alloy-provider = { version = "2.0", features = ["reqwest"] }
alloy-sol-types = "2.0"
alloy-network = { version = "2.0", features = ["ethereum"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
```

### WebSocket Subscriptions

For real-time applications (bots, monitors, event listeners):

```toml
[dependencies]
alloy-primitives = "2.0"
alloy-provider = { version = "2.0", features = ["ws"] }
alloy-sol-types = "2.0"
alloy-network = { version = "2.0", features = ["ethereum"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
futures-util = "0.3"
```

### Hardware Wallet

For applications using Ledger or Trezor:

```toml
[dependencies]
alloy-primitives = "2.0"
alloy-provider = { version = "2.0", features = ["reqwest"] }
alloy-signer-ledger = "2.0"
# or: alloy-signer-trezor = "2.0"
alloy-sol-types = "2.0"
alloy-network = { version = "2.0", features = ["ethereum"] }
```

### Full Suite (All Features)

For complex applications that need everything:

```toml
[dependencies]
alloy = { version = "2.0", features = [
    "full",
    "signer-local",
    "signer-ledger",
    "signer-trezor",
    "provider-http",
    "provider-ws",
    "provider-ipc",
    "contract",
    "network",
    "node-bindings",
] }
tokio = { version = "1", features = ["full"] }
```

## Feature Flags Reference

### alloy (meta-crate) Features

| Feature      | Description                     |
| ------------ | ------------------------------- |
| `full`       | Enables all common features     |
| `consensus`  | Transaction types and consensus |
| `contract`   | High-level contract interaction |
| `network`    | Network abstractions            |
| `providers`  | Provider implementations        |
| `signers`    | All signer implementations      |
| `transports` | HTTP, WS, IPC transports        |
| `sol-types`  | Solidity type bindings          |
| `primitives` | Core primitive types            |

### alloy-provider Features

| Feature   | Description                |
| --------- | -------------------------- |
| `reqwest` | HTTP transport via reqwest |
| `hyper`   | HTTP transport via hyper   |
| `ws`      | WebSocket transport        |
| `ipc`     | IPC transport              |

### alloy-signer Features

| Feature   | Description                      |
| --------- | -------------------------------- |
| `local`   | Private key and mnemonic signers |
| `ledger`  | Ledger hardware wallet           |
| `trezor`  | Trezor hardware wallet           |
| `aws`     | AWS KMS                          |
| `gcp`     | Google Cloud KMS                 |
| `yubihsm` | YubiKey HSM                      |

## ethers-rs to Alloy Migration

### High-Level Mapping

| ethers-rs                          | Alloy                                     | Notes                     |
| ---------------------------------- | ----------------------------------------- | ------------------------- |
| `ethers::providers::Provider`      | `alloy::providers::Provider`              | Different builder pattern |
| `ethers::signers::LocalWallet`     | `alloy::signers::local::PrivateKeySigner` | Different name            |
| `ethers::signers::MnemonicBuilder` | `alloy::signers::local::MnemonicsBuilder` | Similar API               |
| `ethers::contract::Contract`       | `alloy::contract::ContractInstance`       | sol! macro preferred      |
| `ethers::abi::HumanReadableParser` | `sol!` macro                              | Compile-time vs runtime   |
| `ethers::utils::parse_units`       | `alloy::primitives::utils::parse_units`   | Same API                  |
| `ethers::utils::format_units`      | `alloy::primitives::utils::format_units`  | Same API                  |
| `ethers::utils::keccak256`         | `alloy::primitives::keccak256`            | Same function             |
| `ethers::types::Address`           | `alloy::primitives::Address`              | Same type                 |
| `ethers::types::U256`              | `alloy::primitives::U256`                 | Same type                 |
| `ethers::types::Bytes`             | `alloy::primitives::Bytes`                | Same type                 |
| `ethers::types::H256`              | `alloy::primitives::B256`                 | Renamed                   |
| `ethers::types::I256`              | `alloy::primitives::I256`                 | Same type                 |

### Provider Migration

```rust
// ethers-rs
let provider = Provider::<Http>::try_from("https://rpc.example.com")?;
let wallet = "0x...".parse::<LocalWallet>()?.with_chain_id(1u64);
let client = SignerMiddleware::new(provider, wallet);

// Alloy
let signer: PrivateKeySigner = "0x...".parse()?;
let provider = ProviderBuilder::new()
      // Replaces manually adding middleware
    .wallet(wallet)
    .connect_http("https://rpc.example.com".parse()?);
```

Key differences:

- Alloy uses `ProviderBuilder` with a fluent API instead of middleware wrapping
- Fillers (gas, nonce, chain ID) are added via `with_recommended_fillers()` instead of separate middleware
- No need for `SignerMiddleware` — the signer is added directly to the builder

### Contract Migration

```rust
// ethers-rs
let abi = serde_json::from_str::<Abi>(&abi_str)?;
let contract = Contract::new(address, abi, client);
let result: U256 = contract.method("balanceOf", address)?.call().await?;
let tx = contract.method("transfer", (to, amount))?.send().await?;

// Alloy (preferred: sol! macro)
sol! {
    #[sol(rpc)]
    interface IERC20 {
        function balanceOf(address) external view returns (uint256);
        function transfer(address, uint256) external returns (bool);
    }
}

let contract = IERC20::new(address, &provider);
let result = contract.balanceOf(address).call().await?;
let tx = contract.transfer(to, amount).send().await?;
```

### Signing Migration

```rust
// ethers-rs
let wallet: LocalWallet = "0x...".parse()?;
let sig = wallet.sign_message("hello").await?;

// Alloy
let signer: PrivateKeySigner = "0x...".parse()?;
let sig = signer.sign_message(b"hello").await?;
```

### Import Path Changes

```rust
// ethers-rs
use ethers::{types::*, providers::*, signers::*, utils::*};

// Alloy
use alloy::primitives::{Address, U256, B256, Bytes};
use alloy::providers::{Provider, ProviderBuilder};
use alloy::signers::{Signer, local::PrivateKeySigner};
use alloy::sol;
```

### Things That Changed Significantly

1. **Contract interaction**: Use the `sol!` macro instead of runtime ABI parsing. This gives compile-time type safety.

2. **Middleware**: Replaced by layers. Instead of wrapping providers in middleware, use `.layer()` in the builder.

3. **Event decoding**: Uses generated types from `sol!` instead of `AbiDecode`.

4. **Transaction request**: The builder API is slightly different. Use `provider.send_transaction().to().value().finish()` instead of constructing a `TransactionRequest` manually.

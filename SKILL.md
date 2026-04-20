---
name: alloy
description: "Comprehensive guide for building blockchain solutions with the Alloy Rust library (by Alloy-Rs). Use this skill whenever the user wants to: interact with Ethereum or EVM-compatible blockchains using Rust; create digital signatures with private keys, mnemonics, hardware wallets (Ledger/Trezor), or cloud KMS (AWS/GCP/YubiKey); send transactions including EIP-1559, EIP-4844, EIP-7702, and legacy; transfer ERC-20 tokens or native ETH; deploy and interact with smart contracts; verify incoming payments or monitor blockchain events; set up RPC providers (HTTP, WebSocket, IPC); use the sol! macro for type-safe Solidity bindings; work with blockchain primitives (Address, U256, Bytes, hashing); subscribe to blocks, logs, or pending transactions; integrate with local nodes (Anvil, Geth, Reth); handle gas estimation, nonce management, and chain ID fillers; build multicall batching; or migrate from ethers-rs to alloy. Make sure to use this skill when the user mentions alloy-rs, alloy, Ethereum Rust, EVM development, Solidity Rust bindings, blockchain Rust SDK, smart contract interaction, crypto payments, on-chain verification, wallet signing, or any EVM blockchain interaction in Rust."
---

# Alloy — Rust Ethereum & EVM Development Library

Alloy is the next-generation Rust library for interacting with Ethereum and all EVM-compatible blockchains. It is maintained by the Reth team at Paradigm and is the successor to ethers-rs. Alloy provides a modular, type-safe, and performant toolkit for every layer of blockchain interaction: providers, signers, contracts, transactions, and network primitives.

## Crate Architecture

| Crate                 | Purpose                                                                  |
| --------------------- | ------------------------------------------------------------------------ |
| `alloy`               | Meta-crate that re-exports everything — start here for simple projects   |
| `alloy-provider`      | RPC providers (HTTP, WS, IPC) with layered middleware                    |
| `alloy-signer-*`      | Wallet/signer implementations (local, Ledger, Trezor, AWS, GCP, YubiKey) |
| `alloy-network`       | Network abstractions (Ethereum, custom chains)                           |
| `alloy-primitives`    | Core types: `Address`, `U256`, `Bytes`, `FixedBytes`, `B256`             |
| `alloy-sol-types`     | Solidity type system: ABI encode/decode, the `sol!` macro                |
| `alloy-contract`      | High-level contract interaction: deploy, call, events                    |
| `alloy-transport`     | Transport layer: HTTP, WS, IPC connections                               |
| `alloy-consensus`     | Transaction types and consensus logic                                    |
| `alloy-json-rpc`      | JSON-RPC type definitions                                                |
| `alloy-node-bindings` | Local node management: Anvil, Geth, Reth                                 |
| `alloy-chains`        | Chain definitions and ID mappings                                        |

## Quick Start: Minimal Dependency Setup

Add the meta-crate for the fastest setup — it re-exports the most commonly needed items:

```toml
# Cargo.toml
[dependencies]
alloy = { version = "1.0", features = ["full"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
eyre = "0.6"
```

For finer-grained control, use individual crates:

```toml
[dependencies]
alloy-primitives = "1.0"
alloy-sol-types = "1.0"
alloy-provider = { version = "1.0", features = ["reqwest"] }
alloy-signer-local = "1.0"
alloy-consensus = "1.0"
alloy-contract = "1.0"
alloy-network = { version = "1.0", features = ["ethereum"] }
alloy-transport = { version = "1.0", features = ["http", "ws"] }
```

## Quick Reference: Which Guide Do I Need?

The user's task determines which reference file to read:

- **Signing messages & creating digital signatures** -> Read `references/signatures-wallets.md`
- **Sending transactions (ETH & ERC-20 tokens)** -> Read `references/transactions-payments.md`
- **Verifying incoming payments & monitoring transfers** -> Read `references/payment-verification.md`
- **Setting up providers (HTTP/WS/IPC) & middleware layers** -> Read `references/providers-networking.md`
- **Deploying & calling smart contracts, sol! macro** -> Read `references/contracts-abi.md`
- **Subscribing to events, blocks, logs** -> Read `references/subscriptions-events.md`
- **Address, U256, Bytes, hashing utilities** -> Read `references/primitives-types.md`
- **Anvil, Geth, Reth local node integration** -> Read `references/node-bindings.md`
- **Cargo setup, feature flags, ethers-rs migration** -> Read `references/cargo-setup.md`

## Core Patterns at a Glance

### 1. Create a Provider and Send a Transaction

This is the most common starting pattern. Every blockchain interaction flows through a provider.

```rust
use alloy::primitives::address;
use alloy::providers::{Provider, ProviderBuilder};
use alloy::signers::local::PrivateKeySigner;

#[tokio::main]
async fn main() -> eyre::Result<()> {
    // Set up signer from private key
    let signer: PrivateKeySigner = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
        .parse()?;

    // Build provider with signer (HTTP by default)
    let rpc_url = "https://eth.llamarpc.com".parse()?;
    let provider = ProviderBuilder::new()
        .with_recommended_fillers()
        .signer(signer)
        .on_http(rpc_url);

    // Get balance
    let address = signer.address();
    let balance = provider.get_balance(address).await?;
    println!("Balance: {}", balance);

    Ok(())
}
```

### 2. Transfer Native ETH

```rust
use alloy::primitives::{address, U256};
use alloy::providers::{Provider, ProviderBuilder};
use alloy::signers::local::PrivateKeySigner;

let signer: PrivateKeySigner = "0x...".parse()?;
let provider = ProviderBuilder::new()
    .with_recommended_fillers()
    .signer(signer.clone())
    .on_http("https://eth.llamarpc.com".parse()?);

// Build and send the transaction
let tx_hash = provider
    .send_transaction()
    .to(address!("0xRecipientAddress"))
    .value(U256::from(1_000_000_000_000_000_000u128)) // 1 ETH (18 decimals)
    .finish()
    .await?;

// Wait for confirmation
let receipt = tx_hash.get_receipt().await?;
println!("Status: {:?}", receipt.status());
```

### 3. Sign a Message (EIP-191 Personal Sign)

```rust
use alloy::signers::{Signer, local::PrivateKeySigner};

let signer: PrivateKeySigner = "0x...".parse()?;
let message = "Hello, Ethereum!";

// Sign with EIP-191 prefix (personal_sign)
let signature = signer.sign_message(message.as_bytes()).await?;
println!("Signature: 0x{}", signature);

// Verify
let recovered = signature.recover_address_from_msg(message)?;
assert_eq!(recovered, signer.address());
```

### 4. ERC-20 Token Transfer

```rust
use alloy::primitives::{address, U256};
use alloy::sol;

// Define the ERC-20 interface using sol! macro
sol! {
    #[sol(rpc)]
    interface IERC20 {
        function transfer(address to, uint256 amount) external returns (bool);
        function balanceOf(address account) external view returns (uint256);
        function decimals() external view returns (uint8);
        function approve(address spender, uint256 amount) external returns (bool);
        function allowance(address owner, address spender) external view returns (uint256);
    }
}

let token_address = address!("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"); // USDC on mainnet
let token = IERC20::new(token_address, &provider);

// Check balance
let balance = token.balanceOf(signer.address()).call().await?._0;
println!("USDC balance: {}", balance);

// Transfer 100 USDC
let tx_hash = token.transfer(address!("0xRecipient"), U256::from(100_000_000u128))
    .send()
    .await?
    .watch()
    .await?;
```

### 5. Deploy a Contract

```rust
sol! {
    #[sol(abi)]
    contract MyContract {
        constructor(uint256 initial_value);
        function value() external view returns (uint256);
    }
}

// Get the compiled bytecode (from forge build or solc)
let bytecode = hex::decode("6080604052...")?;

let deploy_tx = MyContract::deploy_builder(&provider, U256::from(42))
    .deploy()
    .await?;

let contract_address = deploy_tx.address();
println!("Deployed to: {}", contract_address);

// Interact
let contract = MyContract::new(contract_address, &provider);
let value = contract.value().call().await?._0;
```

## Key Concepts

### Provider Layer Stack

Providers are built in layers. Each layer adds functionality (middleware pattern):

```
Your code
    |
SignerLayer (adds wallet/signer for signing)
    |
GasFiller (auto-estimates gas price)
    |
NonceFiller (manages nonce sequence)
    |
ChainIdFiller (auto-fills chain ID)
    |
Transport (HTTP / WS / IPC)
    |
RPC Node
```

Use `ProviderBuilder::new().with_recommended_fillers()` to get all standard fillers. The order matters — the signer layer must be on top so it can sign after all other fillers have populated their fields.

### Supported Transaction Types

| Type     | EIP  | Key Feature                                      |
| -------- | ---- | ------------------------------------------------ |
| Legacy   | -    | Gas price only                                   |
| EIP-1559 | 1559 | Base fee + priority fee, gas tip market          |
| EIP-4844 | 4844 | Blob transactions (data availability)            |
| EIP-7702 | 7702 | Account abstraction via delegation               |
| EIP-7594 | 7594 | PeerDAS (peer data availability sampling)        |
| Private  | -    | Private transaction pools (mev-share, flashbots) |

### Signer Types

| Signer             | Crate                  | Use Case                              |
| ------------------ | ---------------------- | ------------------------------------- |
| `PrivateKeySigner` | `alloy-signer-local`   | Local private key, testing, scripts   |
| `MnemonicSigner`   | `alloy-signer-local`   | HD wallet from BIP-39 mnemonic phrase |
| `LedgerSigner`     | `alloy-signer-ledger`  | Ledger hardware wallet                |
| `TrezorSigner`     | `alloy-signer-trezor`  | Trezor hardware wallet                |
| `AwsSigner`        | `alloy-signer-aws`     | AWS KMS signing                       |
| `GcpSigner`        | `alloy-signer-gcp`     | Google Cloud KMS signing              |
| `YubiSigner`       | `alloy-signer-yubihsm` | YubiKey HSM signing                   |

All signers implement the `Signer` trait, so they are interchangeable — swap a `PrivateKeySigner` for a `LedgerSigner` without changing any other code.

### The `sol!` Macro

The `sol!` macro generates type-safe Rust bindings from Solidity source. This is the preferred way to define contract interfaces:

```rust
sol! {
    #[sol(rpc)]  // generates both ABI bindings AND an RPC contract struct
    interface IERC20 {
        function balanceOf(address who) external view returns (uint256);
        function transfer(address to, uint256 amount) external returns (bool);
        event Transfer(address indexed from, address indexed to, uint256 value);
    }
}
```

Key attributes:

- `#[sol(rpc)]` — generates an RPC-ready contract struct
- `#[sol(abi)]` — generates only ABI types (no RPC, for deploy)
- `#[derive(Debug, PartialEq)]` — standard Rust derives on generated types

## Common Pitfalls

1. **Missing fillers** — Always use `.with_recommended_fillers()` unless you have a reason not to. Without the gas filler, you must manually set gas price and gas limit. Without the nonce filler, you must track nonces yourself.

2. **Decimals confusion** — ERC-20 tokens use different decimals (USDC = 6, WETH = 18). Always check with `decimals()` call or use `parse_units` / `format_units` utilities.

3. **Chain ID mismatches** — The `ChainIdFiller` handles this automatically. If you skip it, transactions will be rejected. Never hardcode chain IDs.

4. **Blocking in async context** — All provider methods are async. Hardware signers may block briefly. Use `tokio::task::spawn_blocking` if needed.

5. **Integer overflow** — Always use `U256` for on-chain values, not native Rust integers. The `U256::from()` constructor accepts `u64` or `u128`.

## Reference Files

For detailed information on any topic, read the appropriate reference file:

- `references/signatures-wallets.md` — All signer types, EIP-191/EIP-712 signing, message verification, keystore creation, permit signatures, hardware wallet setup
- `references/transactions-payments.md` — EIP-1559/legacy/4844/7702 transactions, ETH transfers, ERC-20 transfers, permit2 flows, gas estimation, private transactions, transaction lifecycle
- `references/payment-verification.md` — Verifying incoming payments, monitoring transaction receipts, parsing transfer events, filtering logs, building payment listeners, reconciliation patterns
- `references/providers-networking.md` — HTTP/WS/IPC providers, provider builder, layers (retry, fallback, logging, delay), batched RPC, embedded consensus, authenticated connections, dynamic providers
- `references/contracts-abi.md` — sol! macro deep dive, contract deployment from artifact/bytecode, static vs dynamic ABI, JSON ABI loading, revert decoding, multi-call batching, library linking
- `references/subscriptions-events.md` — Block subscriptions, log filtering, pending transaction streams, event multiplexer, poll-based log watching, ENS resolution
- `references/primitives-types.md` — Address, B256, Bytes, FixedBytes, U256/U128, Uint/Bint, parse_units/format_units, keccak256, ecrecover, conversion between types
- `references/node-bindings.md` — Anvil (launch, fork, set storage), Geth integration, Reth local instances, foundry test helpers
- `references/cargo-setup.md` — Feature flags table, minimal vs full crate selection, ethers-rs migration guide, type conversion reference, Cargo.toml patterns per use case

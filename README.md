# Alloy Skill

A comprehensive LLM skill for building blockchain solutions with [Alloy](https://github.com/alloy-rs/alloy) — the next-generation Rust library for Ethereum and EVM-compatible chains.

## What It Does

Enables an LLM to accurately develop, debug, and reason about Rust applications that interact with EVM blockchains using the Alloy library. Covers the full stack: from signing a message to deploying contracts and verifying payments on-chain.

## Features Covered

| Category                 | Capabilities                                                                                                                                                                 |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Digital Signatures**   | EIP-191, EIP-712, EIP-2612 permits — all signer types (private key, mnemonic, Ledger, Trezor, AWS KMS, GCP KMS, YubiKey), keystore management                                |
| **Payments**             | Native ETH transfers, ERC-20 token transfers, approve/transferFrom, permit2 gasless flows, gas estimation strategies, urgent/private transactions                            |
| **Payment Verification** | Receipt checking, real-time event log filtering, Transfer event monitoring, payment listener services, block confirmations, reconciliation                                   |
| **Smart Contracts**      | `sol!` macro for type-safe Solidity bindings, contract deployment (bytecode, artifact, Foundry), static vs dynamic ABI, revert decoding, multicall batching, library linking |
| **Networking**           | HTTP / WebSocket / IPC providers, provider builder with layers (retry, fallback, logging, delay), batched RPC, multi-chain, authenticated connections                        |
| **Subscriptions**        | Real-time block, log, and pending transaction streams, event multiplexing, poll-based fallback, ENS resolution                                                               |
| **Primitives**           | Address, U256, B256, Bytes, `parse_units`/`format_units`, keccak256, ecrecover, type conversions                                                                             |
| **Local Nodes**          | Anvil (launch, fork, storage override, impersonation), Geth, Reth — testing patterns                                                                                         |
| **Migration**            | Complete ethers-rs to Alloy migration guide with type mapping and code examples                                                                                              |

## File Structure

```
alloy/
├── SKILL.md                               # Main skill — quick start, core patterns, reference index
└── references/
    ├── signatures-wallets.md              # Signers, EIP-191/712, keystore, hardware wallets
    ├── transactions-payments.md           # All tx types, ETH/ERC-20 transfers, gas control
    ├── payment-verification.md            # Receipt verification, event monitoring, reconciliation
    ├── providers-networking.md            # HTTP/WS/IPC, layers, batched RPC, multi-chain
    ├── contracts-abi.md                   # sol! macro, deployment, ABI encoding, multicall
    ├── subscriptions-events.md            # Block/log/tx subscriptions, ENS, production patterns
    ├── primitives-types.md               # Address, U256, hashing, conversions
    ├── node-bindings.md                   # Anvil, Geth, Reth, testing patterns
    └── cargo-setup.md                     # Dependencies, feature flags, ethers-rs migration
```

## Quick Example

```rust
use alloy::primitives::{address, U256};
use alloy::providers::{Provider, ProviderBuilder};
use alloy::signers::local::PrivateKeySigner;
use alloy::sol;

// Define ERC-20 interface
sol! {
    #[sol(rpc)]
    interface IERC20 {
        function transfer(address to, uint256 amount) external returns (bool);
        function balanceOf(address account) external view returns (uint256);
    }
}

#[tokio::main]
async fn main() -> eyre::Result<()> {
    let signer: PrivateKeySigner = "0x...".parse()?;
    let provider = ProviderBuilder::new()
        .with_recommended_fillers()
        .signer(signer.clone())
        .on_http("https://eth.llamarpc.com".parse()?);

    let token = IERC20::new(
        address!("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"), // USDC
        &provider,
    );

    let balance = token.balanceOf(signer.address()).call().await?._0;
    println!("USDC balance: {}", balance);

    Ok(())
}
```

## Installation

This skill is designed to be installed as an LLM assistant skill. Once installed, it automatically activates when the user discusses Ethereum, EVM, blockchain payments, smart contracts, wallet signing, or Rust blockchain development — even if they do not explicitly mention "alloy."

## Minimum Dependencies

```toml
[dependencies]
alloy = { version = "1.0", features = ["full"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
eyre = "0.6"
```

## License

Created for LLM-assisted blockchain development.

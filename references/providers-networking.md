# Providers & Networking

This guide covers RPC provider setup (HTTP, WebSocket, IPC), the provider builder, middleware layers, batched RPC calls, and advanced provider patterns.

## Provider Overview

A provider is your gateway to the blockchain. It sends JSON-RPC requests to an Ethereum node and returns typed responses. All blockchain interactions go through a provider.

## Quick Setup (HTTP)

The simplest way to create a provider:

```rust
use alloy::providers::{Provider, ProviderBuilder};

// From URL string
let provider = ProviderBuilder::new()
    .connect_http("https://eth.llamarpc.com".parse()?);

// From Url
let rpc_url = url::Url::parse("https://mainnet.infura.io/v3/YOUR_KEY")?;
let provider = ProviderBuilder::new().connect_http(rpc_url);
```

## Provider with Signer and Fillers

Most applications need a signer (to send transactions) and fillers (to auto-populate gas, nonce, chain ID):

```rust
use alloy::providers::{Provider, ProviderBuilder};
use alloy::signers::local::PrivateKeySigner;

let signer: PrivateKeySigner = "0x...".parse()?;

let provider = ProviderBuilder::new()
      // Gas, Nonce, ChainId fillers
    .wallet(wallet)              // Add signing capability
    .connect_http("https://eth.llamarpc.com".parse()?);
```

Recommended fillers are already included when using `ProviderBuilder::new()`. If you need to opt out, use `.disable_recommended_fillers()`.

1. **ChainIdFiller** — queries the node for chain ID and sets it on every transaction
2. **NonceFiller** — manages nonce tracking to prevent replay and conflicts
3. **GasFiller** — estimates gas price and gas limit for each transaction

## WebSocket Provider

WebSocket connections are essential for subscriptions (blocks, logs, pending transactions). HTTP providers can only poll.

```rust
use alloy::providers::ProviderBuilder;
use alloy::transports::http::Http;

// WS provider with reconnect
let ws_url = "wss://eth-mainnet.g.alchemy.com/v2/YOUR_KEY";
let provider = ProviderBuilder::new()

    .wallet(wallet)
    .on_ws(ws_url.parse()?);
```

### WS with Authentication

```rust
// Bearer token auth
let provider = ProviderBuilder::new()
    .on_ws_with_auth(
        "wss://node.example.com/ws".parse()?,
        Auth::Bearer("your-token".into()),
    );
```

## IPC Provider

For lowest latency when running a local node (Geth, Reth) on the same machine:

```rust
let provider = ProviderBuilder::new()
    .connect_ipc("/path/to/geth.ipc")?;
```

## Provider Builder Pattern

The `ProviderBuilder` supports a fluent API for composing layers:

```rust
let provider = ProviderBuilder::new()
    // Layer order: bottom to top (transport is innermost)
    .layer(logging_layer)      // Optional: log all requests
    .layer(retry_layer)        // Optional: retry failed requests

    .wallet(wallet)
    .connect_http(rpc_url);
```

## Layers (Middleware)

Layers wrap the transport and add cross-cutting concerns like retry, logging, and rate limiting.

### Retry Layer

Automatically retries failed RPC requests with exponential backoff:

```rust
use alloy::provider::layers::RetryLayer;

let provider = ProviderBuilder::new()
    .layer(RetryLayer::new(3))  // Max 3 retries

    .wallet(wallet)
    .connect_http(rpc_url);
```

### Fallback Layer

Try multiple RPC endpoints in order. If the first fails, try the next:

```rust
use alloy::provider::layers::FallbackLayer;
use alloy::transports::http::{Http, Client};

let http1 = Http::<Client>::new("https://rpc1.example.com".parse()?);
let http2 = Http::<Client>::new("https://rpc2.example.com".parse()?);
let http3 = Http::<Client>::new("https://rpc3.example.com".parse()?);

let provider = ProviderBuilder::new()
    .layer(FallbackLayer::new(vec![http1, http2, http3]))

    .wallet(wallet)
    .connect_http("https://rpc-primary.example.com".parse()?);
```

### Logging Layer

Log all RPC requests and responses for debugging:

```rust
use alloy::provider::layers::LoggingLayer;

let provider = ProviderBuilder::new()
    .layer(LoggingLayer::default())

    .wallet(wallet)
    .connect_http(rpc_url);
```

### Custom Delay Layer

Add artificial delay between requests (rate limiting):

```rust
use std::time::Duration;
use tower::ServiceBuilder;

let provider = ProviderBuilder::new()
    .layer(tower::ServiceBuilder::new()
        .delay(Duration::from_millis(100)))

    .wallet(wallet)
    .connect_http(rpc_url);
```

## Batched RPC Calls

Execute multiple RPC calls in a single HTTP request for efficiency:

```rust
use alloy::providers::ProviderBuilder;

let provider = ProviderBuilder::new()
    .connect_http("https://eth.llamarpc.com".parse()?);

// Multiple independent queries in one batch
let (block_number, gas_price, balance) = tokio::join!(
    provider.get_block_number(),
    provider.get_gas_price(),
    provider.get_balance(my_address),
);

println!("Block: {}, Gas: {}, Balance: {}", block_number?, gas_price?, balance?);
```

## HTTP with Authentication

Some RPC providers require API keys or authentication headers:

```rust
use alloy::transports::http::{Http, hyper::Request, hyper::header};

let mut url = "https://mainnet.infura.io/v3/YOUR_KEY".parse::<url::Url>()?;

// Add custom headers
let provider = ProviderBuilder::new()
    .connect_http(url);
```

## Multi-Chain Provider

When working with multiple chains, create separate providers:

```rust
let eth_provider = ProviderBuilder::new()

    .wallet(eth_signer)
    .connect_http("https://eth.llamarpc.com".parse()?);

let arb_provider = ProviderBuilder::new()

    .wallet(arb_signer)
    .connect_http("https://arb1.arbitrum.io/rpc".parse()?);

let base_provider = ProviderBuilder::new()

    .wallet(base_signer)
    .connect_http("https://mainnet.base.org".parse()?);
```

## Dynamic Provider (Runtime Chain Selection)

For applications that connect to different chains at runtime:

```rust
use alloy::providers::DynProvider;

fn create_provider(rpc_url: &str) -> DynProvider {
    let provider = ProviderBuilder::new()

        .connect_http(rpc_url.parse().unwrap());

    DynProvider::new(provider)
}
```

## Common Provider Methods

| Method                           | Returns           | Description             |
| -------------------------------- | ----------------- | ----------------------- |
| `get_block_number()`             | `u64`             | Latest block number     |
| `get_block_by_number(n, full)`   | `Option<Block>`   | Block by number         |
| `get_block_by_hash(hash, full)`  | `Option<Block>`   | Block by hash           |
| `get_transaction(hash)`          | `Transaction`     | Transaction details     |
| `get_transaction_receipt(hash)`  | `Option<Receipt>` | Transaction receipt     |
| `get_balance(address)`           | `U256`            | ETH balance             |
| `get_transaction_count(address)` | `u64`             | Nonce                   |
| `get_gas_price()`                | `u128`            | Current gas price       |
| `get_code(address)`              | `Bytes`           | Contract bytecode       |
| `get_storage_at(address, slot)`  | `B256`            | Storage slot value      |
| `call(request)`                  | `Bytes`           | Eth_call (read-only)    |
| `estimate_gas(request)`          | `u128`            | Gas estimation          |
| `send_transaction(request)`      | `PendingTx`       | Broadcast transaction   |
| `get_logs(filter)`               | `Vec<Log>`        | Query event logs        |
| `watch_blocks()`                 | `Stream<B256>`    | Subscribe to new blocks |
| `subscribe_logs(filter)`         | `Stream<Log>`     | Subscribe to events     |

## Error Handling

```rust
use alloy::transports::TransportError;

match provider.get_balance(address).await {
    Ok(balance) => println!("Balance: {}", balance),
    Err(TransportError::ErrorResp(resp)) => {
        eprintln!("RPC error: {} - {}", resp.code, resp.message);
    }
    Err(TransportError::HttpError(e)) => {
        eprintln!("HTTP error: {}", e);
    }
    Err(e) => {
        eprintln!("Other error: {}", e);
    }
}
```

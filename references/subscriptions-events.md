# Subscriptions & Events

This guide covers real-time blockchain monitoring: subscribing to new blocks, filtering event logs, watching pending transactions, and building event-driven applications.

## Overview

Subscriptions require a WebSocket or IPC connection. HTTP providers can only poll (request-response). Always use `.on_ws()` or `.on_ipc()` when you need real-time data.

```rust
use alloy::providers::ProviderBuilder;

// WS is required for subscriptions
let provider = ProviderBuilder::new()
    .on_ws("wss://eth-mainnet.g.alchemy.com/v2/YOUR_KEY".parse()?);
```

## Subscribe to New Blocks

Get notified immediately when a new block is produced:

```rust
use futures_util::StreamExt;

let mut block_stream = provider.watch_blocks().await?;

while let Some(block_hash) = block_stream.next().await {
    println!("New block: {}", block_hash);

    // Get full block details if needed
    let block = provider.get_block_by_hash(block_hash, true.into()).await?;
    if let Some(block) = block {
        println!("  Transactions: {}", block.transactions.len());
        println!("  Gas used: {}", block.gas_used);
        println!("  Timestamp: {}", block.timestamp);
    }
}
```

## Subscribe to Pending Transactions

Monitor the mempool for incoming transactions before they are confirmed:

```rust
use futures_util::StreamExt;

let mut tx_stream = provider.subscribe_pending_transactions().await?;

while let Some(tx_hash) = tx_stream.next().await {
    println!("Pending tx: {}", tx_hash);

    // Get full transaction details
    if let Some(tx) = provider.get_transaction_by_hash(tx_hash).await? {
        if tx.to() == Some(my_address) {
            println!("  Incoming payment of {} ETH from {}!", tx.value(), tx.from());
        }
    }
}
```

## Filter and Subscribe to Logs

Log subscriptions are the primary way to monitor contract events in real-time.

### Basic Log Filter

```rust
use alloy::primitives::{address, Address, B256};
use alloy::rpc::types::Filter;
use futures_util::StreamExt;

let token_address = address!("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48");

// Transfer event signature
let transfer_topic = B256::from_slice(
    &alloy::primitives::keccak256(b"Transfer(address,address,uint256)")
);

// Build filter
let filter = Filter::new()
    .address(token_address)
    .event_signature(transfer_topic);

let mut log_stream = provider.subscribe_logs(&filter).await?;

while let Some(log) = log_stream.next().await {
    println!("Transfer event in tx: {:?}", log.transaction_hash);
}
```

### Filter by Indexed Parameters

Solidity events can have up to 3 indexed parameters, stored as `topic1`, `topic2`, `topic3` in the log:

```rust
// Watch for transfers TO a specific address
let filter = Filter::new()
    .address(token_address)
    .event_signature(transfer_topic)
    .topic1(my_address);  // `to` parameter

// Watch for transfers FROM a specific address
let filter = Filter::new()
    .address(token_address)
    .event_signature(transfer_topic)
    .topic0(from_address); // `from` parameter

// Watch for transfers between specific addresses
let filter = Filter::new()
    .address(token_address)
    .event_signature(transfer_topic)
    .topic0(from_address)
    .topic1(to_address);

// Watch for any value above a threshold (value is NOT indexed, so use data field)
// Note: non-indexed parameters cannot be filtered at the node level — filter in code
```

### Filter by Block Range

For historical queries (not subscriptions):

```rust
let filter = Filter::new()
    .address(token_address)
    .from_block(BlockNumber::Number(18_000_000))
    .to_block(BlockNumber::Number(18_100_000));

let logs = provider.get_logs(&filter).await?;
```

### Multi-Token Log Watcher

```rust
let tokens = vec![
    address!("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"), // USDC
    address!("0xdAC17F958D2ee523a2206206994597C13D831ec7"), // USDT
    address!("0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"), // WETH
];

let filter = Filter::new()
    .address(tokens)
    .event_signature(transfer_topic)
    .topic1(my_address);  // Only transfers TO me
```

## Event Multiplexer

Combine multiple event subscriptions into one stream:

```rust
use futures_util::StreamExt;
use std::collections::HashMap;

let mut block_stream = provider.watch_blocks().await?;

// Create log filters for different event types
let transfer_filter = Filter::new()
    .address(token_address)
    .event_signature(transfer_topic);

let approval_filter = Filter::new()
    .address(token_address)
    .event_signature(approval_topic);

let mut log_stream = provider.subscribe_logs(&transfer_filter).await?;

// Combine into one loop using select
tokio::select! {
    Some(block_hash) = block_stream.next() => {
        println!("New block: {}", block_hash);
    }
    Some(log) = log_stream.next() => {
        println!("Transfer event: {:?}", log.transaction_hash);
    }
}
```

### High-Performance Block Sweeping (The 1-Millisecond Rule)

Never poll the database per transaction or poll individual user addresses. Instead, sweep logs for token contracts and intersect them with an in-memory `HashSet` of user addresses.

```rust
use std::collections::HashSet;
use alloy::primitives::Address;

// 1. Load all million users from DB into RAM once at startup
let my_users: HashSet<Address> = fetch_all_users_from_db().await?;

// 2. Fetch logs for a range of blocks for the USDC contract
let filter = Filter::new()
    .address(usdc_contract)
    .from_block(1000)
    .to_block(1005)
    .event_signature(transfer_topic);

let logs = provider.get_logs(&filter).await?;

// 3. Filter in RAM (takes < 1ms for thousands of logs)
for log in logs {
    let transfer = Transfer::decode_log(&log)?;
    // transfer.to is the recipient (topic2)
    if my_users.contains(&transfer.to) {
        // Send MQ message to credit user's balance
        println!("User received payment: {}", transfer.value);
    }
}
```

## Poll-Based Log Watching (HTTP)

When you cannot use WebSocket, poll for new logs at intervals:

```rust
use alloy::rpc::types::Filter;
use alloy::primitives::BlockNumber;
use tokio::time::{interval, Duration};

let mut last_block = provider.get_block_number().await?;
let mut poll_interval = interval(Duration::from_secs(5));

loop {
    poll_interval.tick().await;
    let current_block = provider.get_block_number().await?;

    if current_block <= last_block {
        continue;
    }

    let filter = Filter::new()
        .address(token_address)
        .from_block(BlockNumber::Number(last_block + 1))
        .to_block(BlockNumber::Number(current_block))
        .event_signature(transfer_topic)
        .topic1(my_address);

    let logs = provider.get_logs(&filter).await?;

    for log in logs {
        println!("Transfer: {:?}", log);
    }

    last_block = current_block;
}
```

## ENS Resolution

ENS (Ethereum Name Service) maps human-readable names to addresses and other records:

```rust
use alloy::providers::Provider;

// Resolve ENS name to address
let address = provider.resolve_name("vitalik.eth").await?;
println!("vitalik.eth -> {}", address);

// Reverse lookup: address to name
let name = provider.lookup_address(address).await?;
println!("{} -> {}", address, name.unwrap_or_default());
```

## Decode Events from Raw Logs

When you receive raw logs and need to decode them:

```rust
use alloy::sol;

sol! {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event Swap(
        address indexed sender,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out,
        address indexed to
    );
}

// Decode from a raw log
let log: alloy::rpc::types::Log = /* ... */;

// Match by topic0 (event signature)
let transfer_sig = alloy::primitives::keccak256(b"Transfer(address,address,uint256)");
let approval_sig = alloy::primitives::keccak256(b"Approval(address,address,uint256)");

    match log.topics.first() {
    Some(topic) if *topic == transfer_sig => {
        let transfer = Transfer::decode_log(&log)?;
        println!("Transfer: {} -> {} = {}", transfer.from, transfer.to, transfer.value);
    }
    Some(topic) if *topic == approval_sig => {
        let approval = Approval::decode_log(&log)?;
        println!("Approval: {} -> {} = {}", approval.owner, approval.spender, approval.value);
    }
    _ => {
        println!("Unknown event");
    }
}
```

## Best Practices for Production Subscriptions

1. **Reconnection**: WS connections can drop. Use the retry layer or implement reconnection logic.

2. **Backpressure**: Process events fast enough, or use a bounded channel to avoid memory issues.

3. **Block confirmation**: For financial applications, wait for N confirmations before acting on events.

4. **Idempotency**: Events may be received multiple times (re-orgs). Track processed event IDs.

5. **Checkpoint**: Save the last processed block number so you can resume after restart:

```rust
// Save checkpoint
let last_processed = log.block_number.unwrap();
std::fs::write("checkpoint.txt", last_processed.to_string())?;

// Resume from checkpoint
let from_block: u64 = std::fs::read_to_string("checkpoint.txt")?.parse()?;
let filter = Filter::new().from_block(BlockNumber::Number(from_block));
```

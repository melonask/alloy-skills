# Payment Verification

This guide covers verifying that incoming payments were received correctly, monitoring for specific transfers, parsing on-chain events, and building robust payment reconciliation systems.

## Verify Transaction Receipt

The most basic verification: check that a transaction succeeded and extract relevant data.

```rust
use alloy::providers::Provider;

let tx_hash: B256 = "0x...".parse()?;

// Get the receipt
let receipt = provider.get_transaction_receipt(tx_hash).await?
    .ok_or_eyre("Transaction not found")?;

// Check success
if receipt.status() {
    println!("Transaction succeeded");
    println!("Gas used: {}", receipt.gas_used);
    println!("Block number: {}", receipt.block_number.unwrap());
    println!("Contract address: {:?}", receipt.contract_address);
} else {
    println!("Transaction REVERTED");
    // Revert reason is encoded in receipt if the node supports it
}
```

## Monitor Incoming ETH Transfers

Watch for ETH transfers to a specific address by subscribing to new blocks and checking internal transactions, or by monitoring the mempool.

### Approach 1: Poll Pending Transactions (Simple)

```rust
use alloy::providers::Provider;
use alloy::primitives::Address;

let my_address: Address = "0xMyAddress".parse()?;
let provider = /* ... */;

// Poll for latest block and check transactions
let block = provider.get_block_by_number(BlockNumber::Latest, false).await?;

if let Some(block) = block {
    for tx in &block.transactions {
        // For full transactions
        if let Some(to) = tx.to() {
            if to == my_address && tx.value() > U256::ZERO {
                println!(
                    "Received {} wei from {} in tx {}",
                    tx.value(),
                    tx.from(),
                    tx.hash
                );
            }
        }
    }
}
```

### Approach 2: Subscribe to New Blocks (Real-time)

```rust
use alloy::providers::{Provider, ProviderBuilder};
use alloy::primitives::Address;
use futures_util::StreamExt;

let my_address: Address = "0xMyAddress".parse()?;

// Subscribe to new block headers
let mut block_stream = provider.watch_blocks().await?;

while let Some(block_hash) = block_stream.next().await {
    let block = provider.get_block_by_hash(block_hash, true.into()).await?;

    if let Some(block) = block {
        for tx in &block.transactions {
            if let Some(to) = tx.to() {
                if to == my_address {
                    println!("Incoming ETH: {} from {}", tx.value(), tx.from());
                }
            }
        }
    }
}
```

## Monitor ERC-20 Transfers via Event Logs

This is the most reliable approach for monitoring token payments. ERC-20 contracts emit `Transfer` events for every transfer.

### Filter Logs for Specific Transfers

```rust
use alloy::providers::{Provider, ProviderBuilder};
use alloy::primitives::{address, Address, Log, B256};
use alloy::sol;

sol! {
    event Transfer(address indexed from, address indexed to, uint256 value);
}

let token_address = address!("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"); // USDC
let my_address: Address = "0xMyAddress".parse()?;

// Get Transfer event signature: keccak256("Transfer(address,address,uint256)")
let transfer_topic = B256::from_slice(
    &alloy::primitives::keccak256(b"Transfer(address,address,uint256)")
);

// Build log filter: watch for Transfer events where `to` is my address
let filter = Filter::new()
    .address(token_address)
    .event_signature(transfer_topic)
    .topic1(my_address);  // `to` is the second indexed parameter (topic1)

// Get past logs
let logs = provider.get_logs(&filter).await?;

for log in logs {
    // Decode the event
    let transfer = Transfer::decode_log(&log, true)?;
    println!(
        "Received {} tokens from {} (tx: {})",
        transfer.value, transfer.from, log.transaction_hash.unwrap()
    );
}
```

### Subscribe to Transfer Events (Real-time)

```rust
use futures_util::StreamExt;

let filter = Filter::new()
    .address(token_address)
    .event_signature(transfer_topic)
    .topic1(my_address);

let mut log_stream = provider.subscribe_logs(&filter).await?;

while let Some(log) = log_stream.next().await {
    let transfer = Transfer::decode_log(&log, true)?;
    println!(
        "NEW PAYMENT: {} tokens from {} in tx {}",
        transfer.value,
        transfer.from,
        log.transaction_hash.unwrap()
    );
}
```

### Monitor Multiple Tokens

Watch for transfers of any token to your address:

```rust
// Watch all Transfer events to your address (any token contract)
let filter = Filter::new()
    .topic1(my_address)  // `to` parameter

// Or watch specific tokens
let tokens = vec![
    address!("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"), // USDC
    address!("0xdAC17F958D2ee523a2206206994597C13D831ec7"), // USDT
    address!("0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"), // WETH
];

let filter = Filter::new()
    .address(tokens)
    .event_signature(transfer_topic)
    .topic1(my_address);
```

## Verify a Specific Payment

Given a transaction hash, verify that it contains the expected payment:

```rust
use alloy::providers::Provider;
use alloy::primitives::{address, U256, B256};

async fn verify_erc20_payment(
    provider: &impl Provider,
    tx_hash: B256,
    expected_token: Address,
    expected_to: Address,
    expected_from: Address,
    expected_amount: U256,
) -> eyre::Result<bool> {
    // Get the receipt
    let receipt = provider.get_transaction_receipt(tx_hash).await?
        .ok_or_eyre("Transaction not found")?;

    // Must be successful
    if !receipt.status() {
        return Ok(false);
    }

    // Get the Transfer event signature
    let transfer_topic = B256::from_slice(
        &alloy::primitives::keccak256(b"Transfer(address,address,uint256)")
    );

    // Check logs for matching Transfer event
    for log in &receipt.inner.logs {
        if log.address != expected_token {
            continue;
        }
        if log.topics.is_empty() || log.topics[0] != transfer_topic {
            continue;
        }

        // Check `from` (topic1) and `to` (topic2)
        let from = Address::from_word(log.topics.get(1).copied().unwrap_or_default());
        let to = Address::from_word(log.topics.get(2).copied().unwrap_or_default());

        if from != expected_from || to != expected_to {
            continue;
        }

        // Decode amount from data
        let amount = U256::from_be_slice(&log.data.data);
        if amount == expected_amount {
            return Ok(true);
        }
    }

    Ok(false)
}
```

## Build a Payment Listener Service

A complete pattern for building a service that listens for incoming payments and triggers callbacks:

```rust
use alloy::providers::{Provider, ProviderBuilder};
use alloy::primitives::{Address, Filter, B256, U256};
use alloy::sol;
use std::collections::HashMap;
use tokio::sync::mpsc;

sol! {
    event Transfer(address indexed from, address indexed to, uint256 value);
}

struct PaymentListener {
    provider: Arc<impl Provider>,
    monitored_tokens: HashMap<Address, u8>,  // token -> decimals
    watched_address: Address,
}

impl PaymentListener {
    async fn start(&self, tx: mpsc::Sender<PaymentEvent>) -> eyre::Result<()> {
        let transfer_topic = B256::from_slice(
            &alloy::primitives::keccak256(b"Transfer(address,address,uint256)")
        );

        let token_addresses: Vec<Address> = self.monitored_tokens.keys().copied().collect();

        let filter = Filter::new()
            .address(token_addresses)
            .event_signature(transfer_topic)
            .topic1(self.watched_address);

        let mut stream = self.provider.subscribe_logs(&filter).await?;

        while let Some(log) = stream.next().await {
            let transfer = Transfer::decode_log(&log, true)?;
            let decimals = self.monitored_tokens.get(&log.address).copied().unwrap_or(18);
            let formatted = alloy::primitives::utils::format_units(transfer.value, decimals)?;

            let event = PaymentEvent {
                token: log.address,
                from: transfer.from,
                to: transfer.to,
                amount: transfer.value,
                formatted_amount: formatted,
                tx_hash: log.transaction_hash.unwrap(),
                block_number: log.block_number.unwrap(),
            };

            tx.send(event).await?;
        }

        Ok(())
    }
}

struct PaymentEvent {
    token: Address,
    from: Address,
    to: Address,
    amount: U256,
    formatted_amount: String,
    tx_hash: B256,
    block_number: u64,
}
```

## Verify Payment with Block Confirmations

For high-value payments, wait for multiple block confirmations before considering a payment final:

```rust
async fn verify_with_confirmations(
    provider: &impl Provider,
    tx_hash: B256,
    required_confirmations: u64,
) -> eyre::Result<bool> {
    let receipt = provider.get_transaction_receipt(tx_hash).await?
        .ok_or_eyre("Not found")?;

    if !receipt.status() {
        return Ok(false);
    }

    let tx_block = receipt.block_number.unwrap();
    let latest_block = provider.get_block_number().await?;

    if latest_block >= tx_block + required_confirmations {
        Ok(true)  // Sufficiently confirmed
    } else {
        Ok(false) // Not enough confirmations yet
    }
}
```

## Payment Reconciliation Pattern

Compare on-chain data against your expected payments:

```rust
async fn reconcile_payments(
    provider: &impl Provider,
    token_address: Address,
    my_address: Address,
    expected_payments: Vec<(Address, U256)>, // (from, amount)
) -> eyre::Result<Vec<PaymentStatus>> {
    let token = IERC20::new(token_address, provider);

    let mut statuses = Vec::new();

    for (from, expected_amount) in expected_payments {
        let actual_balance = token.balanceOf(my_address).call().await?._0;

        // Check logs for specific transfer from this sender
        let filter = Filter::new()
            .address(token_address)
            .topic1(from)
            .topic2(my_address);

        let logs = provider.get_logs(&filter).await?;
        let total_received: U256 = logs.iter()
            .filter_map(|log| {
                let transfer = Transfer::decode_log(log, true).ok()?;
                Some(transfer.value)
            })
            .sum();

        statuses.push(PaymentStatus {
            from,
            expected: expected_amount,
            received: total_received,
            confirmed: total_received >= expected_amount,
        });
    }

    Ok(statuses)
}
```

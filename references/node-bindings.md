# Node Bindings

This guide covers integrating Alloy with local Ethereum nodes: Anvil (Foundry), Geth, and Reth.

## Anvil (Foundry)

Anvil is Foundry's local Ethereum node, perfect for development and testing.

```toml
[dependencies]
alloy-node-bindings = "1.0"
```

### Launch Anvil Programmatically

```rust
use alloy_node_bindings::Anvil;

// Launch with defaults (random accounts, port 8545)
let anvil = Anvil::new().spawn();

println!("RPC URL: {}", anvil.endpoint_url());
println!("Chain ID: {}", anvil.chain_id());

// Access default test accounts (10 accounts with 10000 ETH each)
let accounts = anvil.addresses();
println!("Account 0: {}", accounts[0]);
println!("Private key 0: {}", anvil.keys()[0]);

// Use the provider
let provider = ProviderBuilder::new()

    .wallet(anvil.wallet().unwrap())
    .connect_http(anvil.endpoint_url().parse()?);
```

### Fork Mainnet with Anvil

Test against real mainnet state without spending real ETH:

```rust
let anvil = Anvil::new()
    .fork("https://eth.llamarpc.com")
    .fork_block_number(19_000_000)  // Pin to specific block
    .spawn();

// Now you have mainnet state at block 19M
// All balances, contracts, and storage are available
let vitalik_balance = provider.get_balance(
    address!("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045")
).await?;
```

### Set Storage (Manipulate Contract State)

Override storage slots for testing edge cases:

```rust
use alloy::primitives::{address, B256};

// Set a storage slot on a contract
anvil.set_storage_at(
    contract_address,
    storage_slot,  // B256
    new_value,     // B256
)?;

// Now the contract's storage slot contains the new value
// Useful for testing: set token balances, modify protocol state, etc.
```

### Deploy Contracts on Anvil

```rust
// Anvil provides instant mining — transactions confirm immediately
let deploy_result = MyContract::deploy_builder(&provider, initial_value)
    .deploy()
    .await?;

let contract_address = deploy_result.address();
```

### Anvil Methods

| Method                             | Description                         |
| ---------------------------------- | ----------------------------------- |
| `Anvil::new()`                     | Create builder with defaults        |
| `.fork(url)`                       | Fork a network at latest block      |
| `.fork_block_number(n)`            | Fork at specific block              |
| `.chain_id(id)`                    | Set chain ID                        |
| `.block_time(secs)`                | Set block time interval             |
| `.arg("--...")`                    | Pass any anvil CLI argument         |
| `.spawn()`                         | Launch the node process             |
| `.endpoint_url()`                  | Get RPC URL (http://127.0.0.1:PORT) |
| `.addresses()`                     | Get generated test accounts         |
| `.keys()`                          | Get private keys for test accounts  |
| `.chain_id()`                      | Get chain ID                        |
| `.set_storage_at(addr, slot, val)` | Override contract storage           |

### Anvil Drop (Cleanup)

The Anvil instance automatically kills the process when dropped:

```rust
{
    let anvil = Anvil::new().spawn();
    // ... use anvil ...
} // Anvil process is killed here
```

## Geth

### Launch Geth Programmatically

```rust
use alloy_node_bindings::Geth;

let geth = Geth::new()
    .arg("--dev")          // Dev mode (single validator, instant mining)
    .spawn();

let provider = ProviderBuilder::new()
    .connect_ipc(geth.ipc_path());
```

### Connect to Running Geth Instance

If Geth is already running, connect via IPC or HTTP:

```rust
// IPC (fastest, same machine)
let provider = ProviderBuilder::new()
    .connect_ipc("~/.ethereum/geth.ipc")?;

// HTTP
let provider = ProviderBuilder::new()
    .connect_http("http://127.0.0.1:8545".parse()?);
```

## Reth

### Launch Reth Programmatically

```rust
use alloy_node_bindings::Reth;

let reth = Reth::new()
    .arg("--dev")          // Dev mode
    .spawn();

let provider = ProviderBuilder::new()
    .connect_ipc(reth.ipc_path());
```

## Testing Patterns with Local Nodes

### Pattern: Fresh Anvil Per Test

```rust
#[tokio::test]
async fn test_transfer() {
    let anvil = Anvil::new().spawn();
    let provider = ProviderBuilder::new()

        .wallet(anvil.wallet().unwrap())
        .connect_http(anvil.endpoint_url().parse().unwrap());

    let recipient = anvil.addresses()[1];

    // Deploy token, transfer, verify — all on local anvil
    let token = deploy_token(&provider).await.unwrap();
    let receipt = token.transfer(recipient, amount).send().await.unwrap().watch().await.unwrap();
    assert!(receipt.status());
}
```

### Pattern: Fork Testing

```rust
#[tokio::test]
async fn test_uniswap_swap() {
    // Fork mainnet so Uniswap contracts exist
    let anvil = Anvil::new()
        .fork("https://eth.llamarpc.com")
        .fork_block_number(19_000_000)
        .spawn();

    let provider = ProviderBuilder::new()

        .wallet(anvil.wallet().unwrap())
        .connect_http(anvil.endpoint_url().parse().unwrap());

    // Interact with real Uniswap contracts on forked state
    // The test account has 10000 ETH from Anvil defaults
}
```

### Pattern: Impersonate Accounts

Anvil allows you to impersonate any address, useful for testing with real mainnet holders:

```rust
// Use anvil_impersonateAccount RPC to send transactions as any address
// This is done through the provider's raw RPC call capability

let impersonated = address!("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045"); // vitalik

// Set balance for impersonated account
anvil.set_balance(impersonated, U256::from(1000_ether))?;
```

## Choosing a Local Node

| Node  | Best For                | Start Time | Features                              |
| ----- | ----------------------- | ---------- | ------------------------------------- |
| Anvil | Testing, forks          | Instant    | Fork, impersonation, storage override |
| Geth  | Production-like         | Slow       | Full consensus, real networking       |
| Reth  | Production, Rust native | Medium     | Full consensus, Rust ecosystem        |

For most development and testing, use Anvil. It starts instantly and has powerful testing features like forking and storage manipulation.

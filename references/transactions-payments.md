# Transactions & Payments

This guide covers sending transactions on Ethereum and EVM-compatible chains: native ETH transfers, ERC-20 token transfers, gas estimation, transaction lifecycle, and transaction types.

## Sending Native ETH

### Transfer ETH to Another Address

```rust
use alloy::primitives::{address, U256};
use alloy::providers::{Provider, ProviderBuilder};
use alloy::rpc::types::TransactionRequest;
use alloy::network::TransactionBuilder;
use alloy::signers::local::PrivateKeySigner;
use alloy::network::EthereumWallet;
use eyre::Result;

#[tokio::main]
async fn main() -> Result<()> {
    let signer: PrivateKeySigner = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
        .parse()?;
    let wallet = EthereumWallet::from(signer.clone());

    let provider = ProviderBuilder::new()
        .wallet(wallet)
        .connect_http("https://ethereum-rpc.publicnode.com".parse()?);

    let tx = TransactionRequest::default()
        .to(address!("0x0000000000000000000000000000000000000000"))
        .value(U256::from(1u128))
        .with_input(Bytes::new());

    let pending = provider.send_transaction(tx).await?;
    println!("Pending tx: {}", pending.tx_hash());
    Ok(())
}
```

### EIP-1559 Transaction (Recommended)

EIP-1559 transactions use a base fee + priority fee model:

```rust
let tx = TransactionRequest::default()
    .to(recipient)
    .value(amount)
    .with_max_fee_per_gas(20_000_000_000)
    .with_max_priority_fee_per_gas(1_000_000_000)
    .with_gas_limit(21_000);
```

### Legacy Transaction

```rust
let tx = TransactionRequest::default()
    .to(recipient)
    .value(amount)
    .with_gas_price(10_000_000_000)
    .with_gas_limit(21_000);
```

## ERC-20 Token Transfers

### Check Balance and Transfer

```rust
use alloy::primitives::{address, U256};
use alloy::providers::{Provider, ProviderBuilder};
use alloy::sol;

sol! {
    #[sol(rpc)]
    interface IERC20 {
        function totalSupply() external view returns (uint256);
        function balanceOf(address account) external view returns (uint256);
        function transfer(address recipient, uint256 amount) external returns (bool);
        function allowance(address owner, address spender) external view returns (uint256);
        function approve(address spender, uint256 amount) external returns (bool);
        function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);

        event Transfer(address indexed from, address indexed to, uint256 value);
        event Approval(address indexed owner, address indexed spender, uint256 value);
    }
}

#[tokio::main]
async fn main() -> eyre::Result<()> {
    let provider = ProviderBuilder::new()
        .connect_http("https://ethereum-rpc.publicnode.com".parse()?);

    let usdc = address!("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48");
    let token = IERC20::new(usdc, &provider);

    let holder = address!("0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503");
    let balance = token.balanceOf(holder).call().await?;
    println!("Balance: {}", balance);

    Ok(())
}
```

### Approve and TransferFrom

```rust
// Approve a spender
let tx = token.approve(spender, U256::from(1_000_000u64))
    .send()
    .await?;
let receipt = tx.watch().await?;
```

## Gas Estimation

### Automatic Gas Estimation (via Fillers)

`ProviderBuilder::new()` includes recommended fillers that auto-estimate gas:

```rust
let provider = ProviderBuilder::new()
    .wallet(wallet)
    .connect_http("https://ethereum-rpc.publicnode.com".parse()?);
```

### Manual Gas Estimation

```rust
let gas_estimate = provider.estimate_gas(tx.clone()).await?;
println!("Estimated gas: {}", gas_estimate);
```

## Transaction Lifecycle

```rust
// 1. Build transaction
let tx = TransactionRequest::default()
    .to(recipient)
    .value(amount);

// 2. Send (broadcast to network)
let pending = provider.send_transaction(tx).await?;
println!("Tx hash: {}", pending.tx_hash());

// 3. Wait for inclusion (mine)
let receipt = pending.get_receipt().await?;
println!("Mined in block: {}", receipt.block_number.unwrap());
println!("Status: {}", receipt.status());

// 4. Or use watch() for send + wait in one step
let tx_hash = token.transfer(recipient, amount)
    .send()
    .await?
    .watch()
    .await?;
```

## Transaction Types Reference

| Type     | EIP  | Key Feature                                      |
| -------- | ---- | ------------------------------------------------ |
| Legacy   | -    | Gas price only                                   |
| EIP-1559 | 1559 | Base fee + priority fee, gas tip market          |
| EIP-4844 | 4844 | Blob transactions (data availability)            |
| EIP-7702 | 7702 | Account abstraction via delegation               |

## Common Payment Patterns

### Batch Payments (Multicall)

Use Multicall3 for gas-efficient batch operations:

```rust
sol! {
    #[sol(rpc)]
    interface Multicall3 {
        function aggregate3(Call3[] calldata calls)
            external payable returns (Result[] memory returnData);

        struct Call3 {
            address target;
            bool allowFailure;
            bytes callData;
        }

        struct Result {
            bool success;
            bytes returnData;
        }
    }
}
```

### Checking Token Decimals Before Transfer

```rust
let decimals = token.decimals().call().await?;
let amount = parse_units("100.5", decimals)?;
```

### Handling Insufficient Balance

```rust
let balance = token.balanceOf(signer.address()).call().await?;
if balance < amount {
    return Err(eyre!("Insufficient balance"));
}
```

# Contracts & ABI

This guide covers the `sol!` macro, contract deployment, contract interaction, ABI encoding/decoding, revert handling, and multi-call patterns.

## The `sol!` Macro

The `sol!` macro is Alloy's primary interface for defining Solidity types in Rust. It generates type-safe bindings for function calls, events, errors, structs, and enums.

### Basic Contract Interface

```rust
use alloy::sol;

sol! {
    #[sol(rpc)]
    interface IERC20 {
        function balanceOf(address account) external view returns (uint256);
        function transfer(address to, uint256 amount) external returns (bool);
        function approve(address spender, uint256 amount) external returns (bool);
        function allowance(address owner, address spender) external view returns (uint256);

        event Transfer(address indexed from, address indexed to, uint256 value);
        event Approval(address indexed owner, address indexed spender, uint256 value);
    }
}
```

### Attributes Explained

| Attribute               | Effect                                                                 |
| ----------------------- | ---------------------------------------------------------------------- |
| `#[sol(rpc)]`           | Generates both ABI types and an RPC-ready contract struct with `new()` |
| `#[sol(abi)]`           | Generates only ABI types (no RPC struct — used for deployment)         |
| `#[sol(all_derives)]`   | Adds `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash` derives to all types  |
| `#[derive(Eip712Hash)]` | Enables EIP-712 typed data signing for the struct                      |

### Function Call Patterns

```rust
let token = IERC20::new(token_address, &provider);

// Read-only call (no gas, no on-chain state change)
let balance = token.balanceOf(my_address).call().await?;

// State-changing call (costs gas, changes state)
let pending_tx = token.transfer(recipient, amount)
    .send()          // Broadcast transaction
    .await?;

// Wait for confirmation
let receipt = pending_tx.get_receipt().await?;

// Or watch (sends and waits in one step)
let receipt = token.transfer(recipient, amount)
    .send()
    .await?
    .watch()
    .await?;
```

### Events

```rust
// Emitting events is on-chain (contracts do this).
// In Rust, you decode events from logs:

let logs = provider.get_logs(&filter).await?;
for log in logs {
    let transfer = IERC20::Transfer::decode_log(&log, true)?;
    println!("Transfer: {} -> {} = {}", transfer.from, transfer.to, transfer.value);
}
```

## Contract Deployment

### Deploy from sol! Definition

```rust
sol! {
    #[sol(abi)]
    contract SimpleStorage {
        constructor(uint256 initial_value);
        function value() external view returns (uint256);
        function set_value(uint256 new_value) external;
    }
}

// Compile your contract with `forge build` or `solc`
// The bytecode is in `out/SimpleStorage.sol/SimpleStorage.json`

// Load bytecode (you can embed it or load from file)
let bytecode_bytes = hex::decode("6080604052348015600f57600080fd5b5060405160...")?;
let bytecode = Bytes::from(bytecode_bytes);

// Deploy
let deploy_result = SimpleStorage::deploy_builder(&provider, U256::from(42))
    .deploy()
    .await?;

let contract_address = deploy_result.address();
let receipt = deploy_result.get_receipt().await?;

// Interact with deployed contract
let contract = SimpleStorage::new(contract_address, &provider);
let value = contract.value().call().await?;
assert_eq!(value, U256::from(42));
```

### Deploy from Artifact JSON (Foundry)

```rust
use std::fs;

// Load Foundry artifact
let artifact_json = fs::read_to_string("out/MyContract.sol/MyContract.json")?;
let artifact: serde_json::Value = serde_json::from_str(&artifact_json)?;
let bytecode_hex = artifact["bytecode"]["object"].as_str().unwrap();
let bytecode = hex::decode(bytecode_hex.replace("0x", ""))?;
```

### Deploy with Constructor Arguments

```rust
sol! {
    #[sol(abi)]
    contract Token {
        constructor(string name, string symbol, uint8 decimals_);
        function mint(address to, uint256 amount) external;
    }
}

let deploy_result = Token::deploy_builder(
    &provider,
    "MyToken".to_string(),
    "MTK".to_string(),
    18u8,
)
.deploy()
.await?;
```

## Loading ABI from JSON

For interacting with existing contracts where you have the ABI JSON:

```rust
use alloy::contract::ContractInstance;
use alloy::json_abi::JsonAbi;

// Load ABI
let abi_json = r#"[
    {"type":"function","name":"balanceOf","inputs":[{"name":"account","type":"address"}],"outputs":[{"name":"","type":"uint256"}],"stateMutability":"view"},
    {"type":"function","name":"transfer","inputs":[{"name":"to","type":"address"},{"name":"amount","type":"uint256"}],"outputs":[{"name":"","type":"bool"}],"stateMutability":"nonpayable"}
]"#;

let abi: JsonAbi = serde_json::from_str(abi_json)?;
```

## Static vs Dynamic ABI

Alloy offers two ABI systems:

### Static ABI (sol! macro — Preferred)

Type-safe at compile time. The compiler catches mismatches between your Solidity interface and how you use it in Rust.

```rust
sol! {
    #[sol(rpc)]
    interface MyContract {
        function getValue() external view returns (uint256);
    }
}

// Compile-time type safety
let result = contract.getValue().call().await?; // Returns U256 directly
```

### Dynamic ABI (runtime resolution)

Useful when the ABI is not known at compile time, or when building generic tools like explorers or multisig frontends.

```rust
use alloy::dyn_abi::{DynSolValue, DynSolType};
use alloy::json_abi::{JsonAbi, Function};

// Parse ABI at runtime
let abi: JsonAbi = serde_json::from_str(&abi_json)?;

// Call a function by name
let function: &Function = abi.function("balanceOf").unwrap();

// Encode parameters
let params = vec![
    DynSolValue::Address(my_address),
];

let calldata = function.abi_encode_input(&params)?;

// Make the call
let result_bytes = provider.call().to(contract_address).calldata(calldata.into()).await?;

// Decode output
let outputs = function.abi_decode_output(&result_bytes, false)?;
```

## Revert Decoding

When a transaction reverts, Alloy can decode the revert reason:

```rust
sol! {
    // Define custom errors from the contract
    error InsufficientBalance(uint256 available, uint256 required);
    error Unauthorized();
    error AlreadyExists();
}

// After a reverted transaction
let receipt = pending_tx.get_receipt().await?;
if !receipt.status() {
    // The revert reason is available through the provider
    // You can decode it if you know the ABI
    let result = contract.my_function().call().await;

    match result {
        Err(e) => {
            // Check for known revert reasons
            // The error message will include the revert data
            println!("Revert reason: {}", e);
        }
        Ok(_) => {}
    }
}
```

## Complex Types in sol!

### Structs

```rust
sol! {
    struct Order {
        address maker;
        address tokenIn;
        address tokenOut;
        uint128 amountIn;
        uint128 amountOut;
        uint256 deadline;
    }
}
```

### Enums

```rust
sol! {
    enum Status {
        Pending,
        Active,
        Closed,
    }
}
```

### Custom Errors

```rust
sol! {
    error InsufficientFunds(uint256 required, uint256 available);
    error NotOwner(address caller, address owner);
}
```

### Events with All Data

```rust
sol! {
    event Swap(
        address indexed sender,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out,
        address indexed to
    );
}
```

## Multicall Batching

Bundle multiple contract calls into a single on-chain transaction for gas savings:

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

// Multicall3 is deployed at the same address on every chain:
// 0xcA11bde05977b3631167028862bE2a173976CA11

let multicall = Multicall3::new(
    address!("0xcA11bde05977b3631167028862bE2a173976CA11"),
    &provider,
);

// Build individual calls
let call1 = token_a.balanceOf.encode_call((my_address,));
let call2 = token_b.balanceOf.encode_call((my_address,));

let calls = vec![
    Call3 {
        target: token_a_address,
        allowFailure: false,
        callData: call1,
    },
    Call3 {
        target: token_b_address,
        allowFailure: false,
        callData: call2,
    },
];

let results = multicall.aggregate3(calls).call().await?;

// Decode individual results
let balance_a = U256::from_be_slice(&results[0].returnData);
let balance_b = U256::from_be_slice(&results[1].returnData);
```

## Library Linking

When deploying contracts that use libraries, link the library addresses into the bytecode:

```rust
use alloy::contract::ContractInstance;

// 1. Deploy the library first
let lib_receipt = MyLibrary::deploy_builder(&provider).deploy().await?;
let lib_address = lib_receipt.address();

// 2. Link the library into the main contract bytecode
let linked_bytecode = main_bytecode.replace(
    "__$LIBRARY_PLACEHOLDER$__",
    &format!("{:x}", lib_address),
);

// 3. Deploy the main contract with linked bytecode
let main_receipt = MainContract::deploy_builder(&provider)
    .from(linked_bytecode)
    .deploy()
    .await?;
```

## Interacting with Contract Instances

Once you have a contract instance, all defined functions are available as methods:

```rust
let token = IERC20::new(token_address, &provider);

// View functions (free, no gas)
let balance = token.balanceOf(addr).call().await?;
let decimals = token.decimals().call().await?;
let total_supply = token.totalSupply().call().await?;

// Write functions (cost gas)
let tx = token.approve(spender, amount).send().await?;
let receipt = tx.watch().await?;

// With explicit gas and value
let tx = token.somePayableFunction()
    .gas(100_000)
    .value(U256::from(1_ether))
    .send()
    .await?;
```

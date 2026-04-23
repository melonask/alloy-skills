# Signatures & Wallets

This guide covers all signer types, message signing, EIP-191 personal sign, EIP-712 typed data, signature verification, keystore creation, and hardware wallet setup.

## Private Key Signer

The simplest signer for testing and scripts:

```rust
use alloy::signers::local::PrivateKeySigner;
use alloy::signers::Signer;

let signer: PrivateKeySigner = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
    .parse()?;

let address = signer.address();
println!("Address: {}", address);
```

### Sign a Message (EIP-191)

```rust
let message = "Hello, Ethereum!";
let signature = signer.sign_message(message.as_bytes()).await?;
println!("Signature: 0x{}", signature);

// Verify
let recovered = signature.recover_address_from_msg(message)?;
assert_eq!(recovered, signer.address());
```

### Sign Raw Bytes

```rust
let data = b"raw data to sign";
let signature = signer.sign_message(data).await?;
```

## Wallet Creation

### EthereumWallet from Signer

```rust
use alloy::network::EthereumWallet;
use alloy::signers::local::PrivateKeySigner;

let signer: PrivateKeySigner = "0x...".parse()?;
let wallet = EthereumWallet::from(signer.clone());
```

## Signature Verification

### Recover Address from Message

```rust
use alloy::sol_types::SolEvent;
use alloy::primitives::B256;

let signature = signer.sign_message(b"hello").await?;

// EIP-191 verification
let recovered = signature.recover_address_from_msg("hello")?;
assert_eq!(recovered, signer.address());
```

### Recover from Hash

```rust
use alloy::primitives::{keccak256, B256};

let hash: B256 = keccak256(b"hello");
let recovered = signature.recover_address_from_hash(&hash)?;
```

## EIP-712 Typed Data Signing

EIP-712 provides structured data signing for better UX and security:

```rust
use alloy::sol;
use alloy::signers::local::PrivateKeySigner;

sol! {
    struct MyMessage {
        string content;
        uint256 timestamp;
    }
}

let signer: PrivateKeySigner = "0x...".parse()?;
let message = MyMessage {
    content: "Hello".to_string(),
    timestamp: U256::from(1234567890),
};

// Sign EIP-712 typed data
let domain = alloy::eip712_domain! {
    name: "MyApp",
    version: "1",
    chain_id: 1,
};
let signature = signer.sign_typed_data(&message, &domain).await?;
```

## Keystore File

### Create a Keystore

```rust
use alloy::signers::local::PrivateKeySigner;
use std::fs;

let signer = PrivateKeySigner::random();
let keystore = signer.encrypt_keystore("./keystores", "password")?;
println!("Keystore created: {}", keystore);
```

### Load from Keystore

```rust
let signer = PrivateKeySigner::decrypt_keystore("./keystores/UTC--...", "password")?;
```

## Mnemonic (BIP-39)

### Create from Mnemonic

```rust
use alloy::signers::local::PrivateKeySigner;

let signer = PrivateKeySigner::from_phrase(
    "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about",
    None, // password
)?;
```

### Generate Random Wallet

```rust
let signer = PrivateKeySigner::random();
let address = signer.address();
println!("Random address: {}", address);
```

## Hardware Wallets

### Ledger

```toml
[dependencies]
alloy-signer-ledger = "2.0"
```

```rust
use alloy::signers::ledger::LedgerSigner;

let signer = LedgerSigner::new().await?;
println!("Ledger address: {}", signer.address());

let signature = signer.sign_message(b"test").await?;
```

### Trezor

```toml
[dependencies]
alloy-signer-trezor = "2.0"
```

```rust
use alloy::signers::trezor::TrezorSigner;

let signer = TrezorSigner::new().await?;
```

## Cloud KMS Signers

### AWS KMS

```toml
[dependencies]
alloy-signer-aws = "2.0"
```

```rust
use alloy::signers::aws::AwsSigner;

let signer = AwsSigner::new(
    aws_config,
    key_id,
    Some(chain_id),
).await?;
```

### GCP KMS

```toml
[dependencies]
alloy-signer-gcp = "2.0"
```

```rust
use alloy::signers::gcp::GcpSigner;

let signer = GcpSigner::new(
    project_id,
    location,
    key_ring,
    key_name,
    version,
).await?;
```

## Signer Trait (All Signers Implement This)

```rust
use alloy::signers::Signer;

async fn sign_with_any_signer(signer: &impl Signer, message: &[u8]) -> eyre::Result<()> {
    let signature = signer.sign_message(message).await?;
    let recovered = signature.recover_address_from_msg(message)?;
    assert_eq!(recovered, signer.address());
    Ok(())
}
```

## Signing a Transaction

```rust
use alloy::signers::local::PrivateKeySigner;
use alloy::network::EthereumWallet;
use alloy::providers::ProviderBuilder;
use alloy::primitives::U256;
use alloy::rpc::types::TransactionRequest;
use alloy::network::TransactionBuilder;

let signer: PrivateKeySigner = "0x...".parse()?;
let wallet = EthereumWallet::from(signer.clone());

let provider = ProviderBuilder::new()
    .wallet(wallet)
    .connect_http("https://ethereum-rpc.publicnode.com".parse()?);

let tx = TransactionRequest::default()
    .to(address!("0x..."))
    .value(U256::from(1u128))
    .with_input(Bytes::new());

// The provider signs and broadcasts
let pending = provider.send_transaction(tx).await?;
```

## Permits (EIP-2612)

```rust
sol! {
    interface IERC20Permit {
        function permit(
            address owner,
            address spender,
            uint256 value,
            uint256 deadline,
            uint8 v,
            bytes32 r,
            bytes32 s
        ) external;
    }
}
```

## Test Utilities

### Anvil Test Accounts

```rust
use alloy::node_bindings::Anvil;
use alloy::providers::{Provider, ProviderBuilder};

let anvil = Anvil::new().spawn();
let accounts = anvil.addresses();
let signer: PrivateKeySigner = anvil.keys()[0].parse()?;
```

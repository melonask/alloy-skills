# Known Issues Found During Testing

This file documents real compilation/API errors discovered by testing the skill's code examples against `alloy` crate version `2.0.0` (April 2026). Each issue includes a minimal reproduction and the verified fix.

---

## Issue 1: `Address` → `B256` conversion via `.into()` not implemented

**Location:** `references/primitives-types.md` (around line 202)

**Original (broken):**

```rust
let addr: Address = address!("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045");
let hash: B256 = addr.into();
```

**Error:**

```
error[E0277]: the trait bound `FixedBytes<32>: From<Address>` is not satisfied
```

**Verified Fix:**

```rust
let hash = addr.into_word(); // Returns B256 (zero-padded)
```

**Test project:** `test-projects/test-primitives`

---

## Issue 2: Outdated `ProviderBuilder` API — `.with_recommended_fillers()`, `.signer()`, `.on_http()`

**Locations:** `SKILL.md`, `references/providers-networking.md`, `references/cargo-setup.md`, `references/node-bindings.md`

In Alloy 2.0.0, the provider builder API changed significantly:

- `ProviderBuilder::new()` **already includes recommended fillers** (gas, nonce, chain ID). Calling `.with_recommended_fillers()` again causes a compiler error.
- `.signer(signer)` was replaced with `.wallet(wallet)` which accepts `EthereumWallet`.
- `.on_http(url)` was renamed to `.connect_http(url)`.

**Original (broken) pattern from SKILL.md and providers-networking.md:**

```rust
let provider = ProviderBuilder::new()
    .with_recommended_fillers()
    .signer(signer)
    .on_http(rpc_url);
```

**Error:**

```
error[E0599]: no method named `with_recommended_fillers` found for struct `ProviderBuilder<...>`
error[E0599]: no method named `signer` found for struct `ProviderBuilder<...>`
error[E0599]: no method named `on_http` found for struct `ProviderBuilder<...>`
```

**Verified Fix:**

```rust
use alloy::network::EthereumWallet;
use alloy::signers::local::PrivateKeySigner;
use alloy::providers::{Provider, ProviderBuilder};

let signer: PrivateKeySigner = "0x...".parse()?;
let wallet = EthereumWallet::from(signer);
let provider = ProviderBuilder::new()
    .wallet(wallet)
    .connect_http("https://eth.llamarpc.com".parse()?);
```

**Test project:** `test-projects/test-provider`

---

## Issue 3: `anvil.accounts()` and `.keys()` are renamed

**Location:** `references/node-bindings.md`

In Alloy 2.0.0:

- `anvil.accounts()` → `anvil.addresses()`
- `anvil.keys()` still exists but returns `&[K256SecretKey]` (can be used with `PrivateKeySigner::from(...)`). A more ergonomic API is `anvil.wallet()` which returns `Option<EthereumWallet>`.
- `.on_http` → `.connect_http`
- `.with_recommended_fillers()` is redundant.

**Original (broken):**

```rust
let accounts = anvil.accounts();
let provider = ProviderBuilder::new()
    .with_recommended_fillers()
    .signer(PrivateKeySigner::from(anvil.keys()[0]))
    .on_http(anvil.endpoint_url().parse()?);
```

**Error:**

```
error[E0599]: no method named `accounts` found for struct `AnvilInstance`
error[E0599]: no method named `with_recommended_fillers` found ...
error[E0599]: no method named `signer` found ...
error[E0599]: no method named `on_http` found ...
```

**Verified Fix:**

```rust
let anvil = Anvil::new().spawn();
let addresses = anvil.addresses();
let provider = ProviderBuilder::new()
    .wallet(anvil.wallet().unwrap())
    .connect_http(anvil.endpoint_url());
let balance = provider.get_balance(addresses[0]).await?;
```

**Test project:** `test-projects/test-node-bindings`

---

## Issue 4: `provider.send_transaction().to().value().finish()` builder pattern does not exist

**Location:** `SKILL.md` (Core Patterns section), `references/cargo-setup.md`

The `Provider` trait's `send_transaction` method takes a `TransactionRequest` as an argument, not a fluent builder on the provider itself.

**Original (broken):**

```rust
let tx_hash = provider
    .send_transaction()
    .to(address!("0xRecipientAddress"))
    .value(U256::from(...))
    .finish()
    .await?;
```

**Note:** The `TransactionBuilder` trait does have `.to()`, `.value()`, etc. on the `TransactionRequest` itself, but you still call `provider.send_transaction(tx).await?` to broadcast. The skill text in `cargo-setup.md` explicitly claims: _"Use `provider.send_transaction().to().value().finish()` instead of constructing a `TransactionRequest` manually"_, which is incorrect.

**Verified Fix:**

```rust
use alloy::rpc::types::TransactionRequest;
use alloy::network::TransactionBuilder;

let tx = TransactionRequest::default()
    .to(address!("0x..."))
    .value(U256::from(1_000_000_000_000_000_000u128))
    .with_input::<Bytes>(vec![].into()); // empty calldata for simple transfer

let pending = provider.send_transaction(tx).await?;
// Wait for confirmation and receipt
let receipt = pending.get_receipt().await?;
```

**Test project:** `test-projects/test-contracts`

---

## Issue 5: `token.decimals().call().await?._0` fails on single-value `u8` return types

**Location:** `SKILL.md` (ERC-20 Token Transfer example)

The `sol!` macro generates a return type that is the underlying Rust type for single-value returns, NOT a struct/tuple with `_0` field.

**Original (broken):**

```rust
let decimals = token.decimals().call().await?._0;
```

**Error:**

```
error[E0610]: `u8` is a primitive type and therefore doesn't have fields
```

**Verified Fix:**

```rust
let decimals = token.decimals().call().await?; // returns u8 directly
let balance = token.balanceOf(address).call().await?; // returns U256 directly
```

**Test project:** `test-projects/test-contracts`

---

## Issue 6: Cargo.toml versions should be updated to `2.0`

**Location:** `SKILL.md`, `references/cargo-setup.md`

The skill specifies `version = "1.0"` for all alloy crates, but crates.io shows `2.0.0` as the latest published version as of April 2026. Using `"1.0"` may pull outdated API surface.

**Verified Fix:** Update all `version = "1.0"` entries to `version = "2.0"`.

Test confirmation: `cargo search alloy` returned `alloy = "2.0.0"`.

---

## Summary

| Issue | Skill File(s)                                                               | API Change                                                                                      |
| ----- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| 1     | `references/primitives-types.md`                                            | `addr.into()` → `addr.into_word()`                                                              |
| 2     | `SKILL.md`, `providers-networking.md`, `cargo-setup.md`, `node-bindings.md` | `.with_recommended_fillers().signer(s).on_http()` → `.wallet(s).connect_http()`                 |
| 3     | `references/node-bindings.md`                                               | `.accounts()`/`.keys()[0]` → `.addresses()`/`.wallet()`; `.on_http` → `.connect_http`           |
| 4     | `SKILL.md`, `cargo-setup.md`                                                | `provider.send_transaction().to()...finish()` → `provider.send_transaction(TransactionRequest)` |
| 5     | `SKILL.md`                                                                  | `call().await?._0` on single-value → `call().await?` directly                                   |
| 6     | All Cargo examples                                                          | version `"1.0"` → `"2.0"`                                                                       |

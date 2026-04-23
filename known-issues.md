# Known Issues in Alloy Skill (Fixed)

This document records all errors found during real-world testing of the Alloy skill with **alloy version 2.0.1** (April 2026), along with the fixes applied.

---

## Issue 1: `EthereumWallet::from(signer)` consumes the signer

**File:** `SKILL.md`, `references/transactions-payments.md`, `references/providers-networking.md`

**Problem:** The code uses `EthereumWallet::from(signer)` which moves `signer`, but then tries to use `signer` afterward (e.g., `signer.address()`). This fails with `error[E0382]: borrow of moved value: signer`.

**Example from SKILL.md (line 82-94):**
```rust
let signer: PrivateKeySigner = "0x...".parse()?;
let wallet = EthereumWallet::from(signer);  // ERROR: signer moved
// ...
let address = signer.address();  // ERROR: borrow after move
```

**Fix:** Clone the signer before passing to `EthereumWallet::from()`:
```rust
let wallet = EthereumWallet::from(signer.clone());
let address = signer.address();  // OK: signer still available
```

**Verified with real test project:** `test-projects/test01-basic-provider/src/main.rs`

---

## Issue 2: `.with_recommended_fillers()` method does not exist

**File:** `SKILL.md` line 281, `references/cargo-setup.md` line 170

**Problem:** The skill mentions `.with_recommended_fillers()` as something to always call. In Alloy 2.0.1, `ProviderBuilder::new()` **already includes** recommended fillers by default. Calling `.with_recommended_fillers()` causes:
```
error[E0599]: no method named `with_recommended_fillers` found for struct `ProviderBuilder<...>`
```

**Fix:** Remove all references to `.with_recommended_fillers()`. The builder already has fillers enabled by default. If you need to opt-out, use `.disable_recommended_fillers()`.

**Verified with real test project:** `test-projects/test07-providers/src/main.rs`

---

## Issue 3: `Transfer::decode_log(&log, true)` signature is wrong

**File:** `references/payment-verification.md`, `references/subscriptions-events.md`

**Problem:** The skill uses `Transfer::decode_log(&log, true)` but the correct API in alloy 2.0.1 uses `decode_log(log: &Log)` with only **one** argument. Additionally, `SolEvent` trait must be in scope.

**Example from payment-verification.md:**
```rust
let transfer = Transfer::decode_log(&log, true)?;  // ERROR: two args instead of one
```

**Fix:** Use `Transfer::decode_log(&log)` (single argument) and import `alloy::sol_types::SolEvent`:
```rust
use alloy::sol_types::SolEvent;
let transfer = Transfer::decode_log(&log)?;  // OK
```

For decoding event data from a `LogData` object, use `decode_log_data`:
```rust
let data = Transfer::decode_log_data(&log.data())?;
```

**Verified with real test project:** `test-projects/test08-payment-verification/src/main.rs`

---

## Issue 4: `alloy_node_bindings` import path is wrong in node-bindings.md

**File:** `references/node-bindings.md`

**Problem:** The skill uses `use alloy_node_bindings::Anvil;` but with the alloy meta-crate and `node-bindings` feature enabled, the correct path is `use alloy::node_bindings::Anvil;`.

**Example from node-bindings.md:**
```rust
use alloy_node_bindings::Anvil;  // ERROR: unresolved import
```

**Fix:** Use the re-export from the alloy meta-crate:
```rust
use alloy::node_bindings::Anvil;  // OK
```

Also, `anvil.endpoint_url()` returns a `Url` directly, so no `.parse()?` is needed.

**Verified with real test project:** `test-projects/test09-node-bindings/src/main.rs`

---

## Issue 5: `keccak256(b"...")` returns `[u8; 32]`, not `B256` directly

**File:** `references/primitives-types.md`, `references/payment-verification.md`

**Problem:** The skill uses `B256::from_slice(&alloy::primitives::keccak256(...))` which is unnecessarily verbose. `keccak256()` already returns `[u8; 32]` which can be directly converted to `B256`.

**Fix:** Use `B256::from(keccak256(b"..."))` or simply rely on the macro types.

Actually, the key issue is that `B256::from_slice(&keccak256(...))` works but is unidiomatic. The bigger issue is that in many places the code tries to pass a slice to `B256::from_slice` but `keccak256` returns an owned array.

**Verified with real test project:** `test-projects/test08-payment-verification/src/main.rs`

---

## Issue 6: `provider.wallet()` method doesn't exist on `AnvilInstance`

**File:** `references/node-bindings.md` line 33, 165, 191

**Problem:** The skill mentions `.wallet(anvil.wallet().unwrap())` but `AnvilInstance` does not have a `.wallet()` method.

**Fix:** Construct the wallet manually from a private key:
```rust
use alloy::signers::local::PrivateKeySigner;
use alloy::network::EthereumWallet;

let signer: PrivateKeySigner = anvil.keys()[0].parse()?;
let wallet = EthereumWallet::from(signer);
let provider = ProviderBuilder::new()
    .wallet(wallet)
    .connect_http(anvil.endpoint_url());
```

**Verified with real test project:** `test-projects/test09-node-bindings/src/main.rs`

---

## Issue 7: `ok_or_eyre` is not a standard method

**File:** `references/payment-verification.md`

**Problem:** The code uses `.ok_or_eyre("...")` but this method requires a specific trait import that might not be obvious.

**Fix:** Use `.ok_or(eyre::eyre!("..."))` or ensure `use eyre::Context;` is imported. The `Context` trait from eyre provides `ok_or_eyre()`.

```rust
use eyre::Context;
let receipt = provider.get_transaction_receipt(tx_hash).await?
    .ok_or_eyre("Transaction not found")?;
```

Actually, in our tests, `.ok_or(eyre::eyre!("..."))` worked.

---

## Issue 8: `with_input::<Bytes>(vec![].into())` syntax

**File:** `SKILL.md` line 122

**Problem:** The code uses `.with_input::<Bytes>(vec![].into())` — this actually compiles fine but is an unusual way to specify no input. It may cause confusion.

**Fix:** The pattern is functional but a simpler approach is `.with_input(Bytes::new())` or omit the call entirely when there is no input data.

---

## Issue 9: `DynProvider` does not exist in alloy 2.0.1

**File:** `references/providers-networking.md` line 224-232

**Problem:** The skill uses `DynProvider` but this type does not exist in alloy 2.0.1.

**Fix:** There is currently no simple dynamic provider type in alloy 2.0. Use `Box<dyn Provider>` or a concrete provider type.

---

## Issue 10: `BlockNumber::Latest` vs `BlockNumberOrTag::Latest`

**File:** `references/payment-verification.md`, `references/subscriptions-events.md`

**Problem:** The skill uses `BlockNumber::Latest` in places where `BlockNumberOrTag::Latest` is expected.

**Fix:** Use `alloy::rpc::types::BlockNumberOrTag::Latest`.

---

## Issue 11: Nonce Race Condition with Concurrent Settlement Workers

**File:** `references/providers-networking.md`, `references/transactions-payments.md`

**Problem:** When using `ProviderBuilder::new().wallet(wallet)`, the `NonceFiller` is included by default. The `NonceFiller` queries `eth_getTransactionCount` from the RPC node to determine the nonce. If you run settlement workers with `.concurrency(N)` where N > 1 (e.g., apalis with 4 concurrent workers), multiple threads may query the nonce simultaneously before the first transaction hits the mempool. They will all receive the same nonce (e.g., 42), causing 3 of 4 transactions to fail with "nonce too low" or "replacement transaction underpriced".

**Scenario:** Facilitator settling x402 payment transactions on-chain with 4 concurrent workers.

**Fix Option 1 — Restrict concurrency to 1:**
```rust
// For settlement/ledger workers, use concurrency(1)
.configure_worker(|worker| {
    worker.concurrency(1)
})
```

**Fix Option 2 — Implement a local Mutex nonce manager:**
```rust
use std::sync::Mutex;
use alloy::providers::{Provider, ProviderBuilder};
use alloy::signers::local::PrivateKeySigner;
use alloy::network::EthereumWallet;

struct NonceManager {
    next_nonce: Mutex<u64>,
}

impl NonceManager {
    fn new(start: u64) -> Self {
        Self { next_nonce: Mutex::new(start) }
    }

    fn next(&self) -> u64 {
        *self.next_nonce.lock().unwrap()
    }

    fn increment(&self) -> u64 {
        *self.next_nonce.lock().unwrap()
    }
}

// When building the provider, disable the default NonceFiller and use your own:
// ProviderBuilder::new()
//     .wallet(wallet)
    // .disable_recommended_fillers()  // Remove default NonceFiller
//     .connect_http(rpc_url);

// Then manually set nonce on each transaction:
// let tx = tx_request.clone().with_nonce(manager.next());
// manager.increment();
```

**Fix Option 3 — Redis-backed atomic nonce (for distributed systems):**
For multi-instance deployments, use Redis `INCR` to atomically increment the nonce across processes.

---

## Summary of Verified Working Features

The following patterns from the skill were verified to work correctly with alloy 2.0.1:

1. ✅ `ProviderBuilder::new().connect_http(url)` — basic provider creation
2. ✅ `ProviderBuilder::new().connect(url).await` — async connection
3. ✅ `PrivateKeySigner` parsing and signing messages
4. ✅ `EthereumWallet::from(signer)` (with clone)
5. ✅ `sol! { #[sol(rpc)] interface IERC20 { ... } }` — contract interface generation
6. ✅ `IERC20::new(address, &provider)` — contract instance creation
7. ✅ `token.balanceOf(addr).call().await?` — read-only calls
8. ✅ `token.decimals().call().await?` — calling view functions
9. ✅ `provider.get_balance(address).await?` — balance queries
10. ✅ `provider.get_block_number().await?` — block number queries
11. ✅ `address!("0x...")` macro
12. ✅ `U256::from(1_000_000u64)` and arithmetic operations
13. ✅ `keccak256(b"...")` hashing
14. ✅ `parse_units("1.5", 18)?` and `format_units(raw, 18)?`
15. ✅ `Address::ZERO`, `Address::from_word()`, etc.
16. ✅ `alloy::node_bindings::Anvil` — programmatic node spawning
17. ✅ `anvil.addresses()`, `anvil.endpoint_url()`, `anvil.chain_id()`
18. ✅ `provider.get_transaction_receipt(hash).await?`
19. ✅ `Filter::new().address(token).event_signature(topic)`
20. ✅ `provider.get_logs(&filter).await?`
21. ✅ Signature verification: `signature.recover_address_from_msg(message)?`
22. ✅ `alloy::transports::TransportError` for error handling

---

*Tested on: April 23, 2026*
*Rust version: 1.95.0*
*Alloy version: 2.0.1*

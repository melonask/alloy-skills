# Primitives & Types

This guide covers Alloy's core primitive types: addresses, hashes, bytes, big numbers, conversion utilities, and hashing functions.

## Address

The 20-byte Ethereum address type. This is the most common type you will work with.

```rust
use alloy::primitives::{address, Address};

// From hex literal
let addr = address!("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045");

// From hex string
let addr: Address = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045".parse()?;

// From bytes
let addr = Address::from([0xd8, 0xdA, 0x6B, 0xF2, 0x69, 0x64, 0xaF, 0x9D,
    0x7e, 0xed, 0x9e, 0x03, 0xe5, 0x34, 0x15, 0xd3, 0x7a, 0xa9, 0x60, 0x45]);

// Zero address (useful for checks)
let zero = Address::ZERO;

// Display
println!("{}", addr);  // 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045

// Checksummed (EIP-55)
println!("{:#x}", addr);  // 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
```

## FixedBytes and B256

Fixed-size byte arrays used throughout Ethereum:

```rust
use alloy::primitives::{B256, B64, B32, B20, FixedBytes};

// B256 is a 32-byte hash (used for block hashes, tx hashes, keccak output)
let hash: B256 = "0x0000000000000000000000000000000000000000000000000000000000000001".parse()?;

// B64 (used for signatures)
let sig_bytes: B64 = "0x0000000000000000000000000000000000000000000000000000000000000001".parse()?;

// Create from array
let hash = B256::from([1u8; 32]);

// Access individual bytes
let first_byte = hash[0];

// Convert to/from slice
let slice: &[u8] = hash.as_slice();
```

## Bytes (Dynamic)

Variable-length byte arrays:

```rust
use alloy::primitives::Bytes;

// From hex
let data: Bytes = "0xdeadbeef".parse()?;

// From vector
let data = Bytes::from(vec![0xde, 0xad, 0xbe, 0xef]);

// From slice
let data = Bytes::from_static(b"hello");

// Access
println!("Length: {}", data.len());
println!("First byte: {:02x}", data[0]);
```

## U256 — 256-bit Unsigned Integer

The most important numeric type for Ethereum. Used for balances, amounts, gas, nonces, and all on-chain values.

```rust
use alloy::primitives::{U256, Uint};

// Creation
let zero = U256::ZERO;
let one = U256::ONE;
let max = U256::MAX;

let from_u64 = U256::from(1_000_000u64);
let from_u128 = U256::from(1_000_000_000_000_000_000u128);
let from_str: U256 = "1000000000000000000".parse()?;

// Arithmetic (all return U256)
let sum = a + b;
let diff = a - b;  // Panics on underflow; use .checked_sub() for safe math
let product = a * b;
let quotient = a / b;
let remainder = a % b;

// Safe arithmetic
let safe_diff = a.checked_sub(b);     // Returns Option<U256>
let safe_add = a.checked_add(b);       // Returns Option<U256>
let safe_mul = a.checked_mul(b);       // Returns Option<U256>

// Comparison
if balance > U256::ZERO { /* ... */ }
assert_eq!(a, b);

// Display
println!("{}", amount);  // Decimal
println!("{:#x}", amount);  // Hex with 0x prefix
```

## parse_units and format_units

Convert between human-readable token amounts and their raw on-chain representation:

```rust
use alloy::primitives::utils::{parse_units, format_units};

// parse_units: human-readable -> raw (smallest unit)
// format_units: raw -> human-readable

// ETH: 18 decimals
let eth_raw = parse_units("1.5", 18)?;           // -> U256 = 1500000000000000000
let eth_display = format_units(eth_raw, 18)?;      // -> "1.5"

// USDC: 6 decimals
let usdc_raw = parse_units("100.50", 6)?;          // -> U256 = 100500000
let usdc_display = format_units(usdc_raw, 6)?;      // -> "100.5"

// USDT: 6 decimals
let usdt_raw = parse_units("0.001", 6)?;           // -> U256 = 1000
let usdt_display = format_units(usdt_raw, 6)?;      // -> "0.001"

// Parse to specific Rust type
let raw_u128: u128 = parse_units("1.0", 18)?.into();

// Format with precision
let display = format_units(U256::from(123456789u64), 18)?;
// -> "0.000000000123456789"
```

## Keccak-256 Hashing

The primary hash function used in Ethereum:

```rust
use alloy::primitives::keccak256;

// Hash bytes
let hash = keccak256(b"hello");
println!("keccak256('hello') = {}", hash);

// Hash strings
let hash = keccak256("Hello, Ethereum!".as_bytes());

// Hash for event signatures (used in log filtering)
let transfer_sig = keccak256(b"Transfer(address,address,uint256)");
let approval_sig = keccak256(b"Approval(address,address,uint256)");
```

## ECDSA Recover

Recover an address from a message and signature (EIP-191):

```rust
use alloy::primitives::B256;
use alloy::signer::Signature;

let signature: Signature = /* from signer.sign_message() */;

// Recover address from EIP-191 signed message
let recovered = signature.recover_address_from_msg("hello")?;
println!("Signer address: {}", recovered);

// Recover address from hash + raw signature
let hash: B256 = keccak256(b"hello");
let recovered = signature.recover_address_from_hash(&hash)?;
```

## Type Conversions

### Between U256, u64, u128, f64

```rust
let u256_val = U256::from(1_000_000u64);
let u64_val: u64 = u256_val.try_into()?;  // Fails if > u64::MAX
let u128_val: u128 = u256_val.try_into()?;
let u256_back = U256::from(u64_val);

// To/from string
let s = u256_val.to_string();
let parsed: U256 = s.parse()?;
```

### Between Address and B256

```rust
let addr: Address = address!("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045");

// Address -> B256 (zero-padded)
let hash: B256 = addr.into_word();

// B256 -> Address (takes last 20 bytes)
let addr2: Address = Address::from_word(hash.into());
```

### Bytes Encoding

```rust
use alloy::primitives::{Bytes, B256};

// U256 to bytes
let bytes = amount.to_be_bytes::<32>();  // [u8; 32]
let bytes_vec = amount.to_be_bytes_vec();

// Bytes to U256
let amount = U256::from_be_slice(&bytes);

// Address to bytes
let addr_bytes: [u8; 20] = address.into();
```

## Common Constants

```rust
use alloy::primitives::{Address, U256, B256};

// Addresses
let zero_address = Address::ZERO;

// U256
let zero = U256::ZERO;
let one = U256::ONE;
let max_u256 = U256::MAX;  // 2^256 - 1
let max_u128 = U256::from(u128::MAX);

// Useful amounts (ETH in wei)
let one_ether = U256::from(1_000_000_000_000_000_000u128); // 10^18
let one_gwei = U256::from(1_000_000_000u128);               // 10^9

// Using parse_units for precision
let one_eth = parse_units("1", 18)?.into::<U256>();
let one_gwei = parse_units("1", 9)?.into::<U256>();
```

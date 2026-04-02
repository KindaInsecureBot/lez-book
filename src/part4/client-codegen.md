# Client Code Generation

## What Is spel-client-gen?

`spel-client-gen` reads your program's IDL and generates typed client code. This is how you integrate LEZ programs into applications.

```bash
spel-client-gen --idl counter.json --out-dir ./client/
```

## Generated Rust Client

The generated Rust client gives you typed function calls:

```rust
// Generated code (don't edit directly)
pub struct CounterClient {
    rpc_url: String,
    program_id: [u8; 32],
}

impl CounterClient {
    pub async fn initialize(
        &self,
        counter: [u8; 32],
        owner: [u8; 32],
        signer_keypair: &Keypair,
    ) -> Result<TxHash, ClientError> {
        // borsh encoding, account ordering, RPC call all handled
    }

    pub async fn increment(
        &self,
        counter: [u8; 32],
        caller: [u8; 32],
        signer_keypair: &Keypair,
    ) -> Result<TxHash, ClientError> {
        // ...
    }
}
```

## C FFI Generation

For integrating with non-Rust applications (e.g., C, Swift, Kotlin via FFI):

```bash
spel-client-gen --idl counter.json --out-dir ./ffi/ --target c
```

Generated C header:

```c
// counter_client.h (generated)
int32_t counter_initialize(
    const uint8_t* counter_id,
    const uint8_t* owner_id,
    const uint8_t* signer_key,
    uint8_t* tx_hash_out
);

int32_t counter_increment(
    const uint8_t* counter_id,
    const uint8_t* caller_id,
    const uint8_t* signer_key,
    uint8_t* tx_hash_out
);
```

## Building Shared Libraries

To build a `.so` or `.dylib` for FFI use:

```toml
# Cargo.toml for your FFI wrapper
[lib]
crate-type = ["cdylib"]

[dependencies]
counter-client = { path = "./client" }
```

```bash
cargo build --release
# Output: target/release/libcounter_client.so
```

## Integration Pattern: Qt/C++ Application

From logos-land (which had a Qt frontend):

```cpp
// Load the generated library
QLibrary clientLib("./liblogosland_client");
clientLib.load();

// Get function pointers
auto claimFn = (ClaimFn)clientLib.resolve("logos_land_claim");

// Call it
uint8_t tx_hash[32];
int result = claimFn(tile_id, player_id, signer_key, x, y, tx_hash);
```

## Account Fetching

The generated client also includes account reading:

```rust
// Read a counter account
let state: CounterState = client.get_counter(&counter_id).await?;
println!("Count: {}", state.count);
```

## Working with the IDL Directly

For custom integrations, you can work with the IDL directly:

```rust
use serde_json::Value;
use borsh::BorshSerialize;

let idl: Value = serde_json::from_str(include_str!("counter.json"))?;

// Build instruction manually
let instruction = build_instruction(
    &idl,
    "increment",
    vec![counter_id, caller_id],
    &increment_args,
)?;
```

# Your First Program: Counter

This chapter walks through building a Counter program on LEZ from scratch. We'll show every command, explain every file, and compare the approach to the Solidity equivalent so you can map your existing mental model onto LEZ's account-based, ZK-proved execution model.

---

## What We're Building

A counter with three operations:

1. **initialize** — create a new counter account owned by an authority
2. **increment** — add one to the count (only the owner can do this)
3. **get_count** — read the current count

### The Solidity Version

Here's the equivalent in Solidity, as a reference point:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Counter {
    uint64 public count;
    address public owner;

    constructor(address _owner) {
        owner = _owner;
        count = 0;
    }

    function increment() external {
        require(msg.sender == owner, "not owner");
        count++;
    }

    function getCount() external view returns (uint64) {
        return count;
    }
}
```

> **🔄 Coming from Solidity?** In Solidity, deploying this contract creates a single instance at a specific address. Each deployment is a separate contract. In LEZ, your program is deployed once by its ImageID — a content hash of the zkVM binary. You don't deploy "instances." Instead, you create **state accounts** that store the counter data, and your single deployed program processes transactions against any of those accounts. Multiple counters = multiple state accounts, all processed by the same program binary.

---

## Step 1: Scaffold the Project

```bash
lez-cli init counter
cd counter
```

This generates a workspace with the following structure:

```
counter/
├── Cargo.toml                              # workspace manifest
├── Makefile                                # build, idl, cli, deploy, inspect targets
├── counter_core/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                          # shared types (state structs, serialization)
├── methods/
│   ├── Cargo.toml
│   ├── build.rs                            # risc0_build::embed_methods()
│   └── guest/
│       └── src/
│           └── bin/
│               └── counter.rs             # on-chain program (NOT main.rs)
└── examples/
    └── src/
        └── bin/
            ├── generate_idl.rs            # IDL generator (uses generate_idl! macro)
            └── counter_cli.rs             # CLI wrapper (3 lines)
```

**Component roles:**

- **`counter_core/`** — shared types used by both the guest and the IDL generator. Put your state struct definitions and manual serialization helpers here. Note: the crate is named `counter_core`, not `core`.
- **`methods/guest/src/bin/counter.rs`** — the actual on-chain program. This is NOT `main.rs` — it lives in `bin/`. Contains `#![no_main]`, `risc0_zkvm::guest::entry!(main)`, and your `#[lez_program]` module.
- **`examples/src/bin/generate_idl.rs`** — generates the IDL JSON. Run via `make idl` or `cargo run --example generate_idl`.
- **`examples/src/bin/counter_cli.rs`** — 3-line CLI wrapper that `lez-cli` uses to dispatch instruction calls.

> **⚠️ Warning:** The scaffold from `lez-cli init` may be missing `risc0-zkvm` metadata in `methods/Cargo.toml`. If you get build errors about missing zkVM crate features or a missing `risc0` dependency, open `methods/Cargo.toml` and verify that the risc0 dependencies are present and have the correct feature flags for your toolchain version.

---

## Step 2: Define the State

Open `counter_core/src/lib.rs` and define the state type for a counter account.

**Important:** `borsh_derive` proc macros **do not compile** for the `riscv32im` guest target. Use manual serialization instead:

```rust
// counter_core/src/lib.rs

/// Counter state: 8 bytes count + 32 bytes owner = 40 bytes total
pub struct CounterState {
    pub count: u64,
    pub owner: [u8; 32],
}

impl CounterState {
    pub const SIZE: usize = 8 + 32;

    pub fn to_bytes(&self) -> Vec<u8> {
        let mut buf = Vec::with_capacity(Self::SIZE);
        buf.extend_from_slice(&self.count.to_le_bytes());
        buf.extend_from_slice(&self.owner);
        buf
    }

    pub fn from_bytes(data: &[u8]) -> Option<Self> {
        if data.len() < Self::SIZE {
            return None;
        }
        let count = u64::from_le_bytes(data[..8].try_into().ok()?);
        let owner: [u8; 32] = data[8..40].try_into().ok()?;
        Some(Self { count, owner })
    }
}
```

> **⚠️ Warning:** `borsh_derive` does **not** compile for the `riscv32im` guest target. Do not use `#[derive(BorshSerialize, BorshDeserialize)]` in code that runs in the guest. Always use manual serialization as shown above. This is one of the most common compilation errors for new LEZ developers.

---

## Step 3: Write the Program Logic

In `methods/guest/src/bin/counter.rs`, define the program with its instructions. Note the required `#![no_main]` and `risc0_zkvm::guest::entry!(main)` — these are mandatory for the RISC Zero guest binary:

```rust
// methods/guest/src/bin/counter.rs
#![no_main]
use counter_core::CounterState;
use nssa_core::account::AccountWithMetadata;
use nssa_core::program::AccountPostState;
use lez_framework::prelude::*;
risc0_zkvm::guest::entry!(main);

#[lez_program]
mod counter {
    #[allow(unused_imports)]
    use super::*;

    /// Initialize a new counter account owned by `authority`.
    #[instruction]
    pub fn initialize(
        #[account(init)]
        counter: AccountWithMetadata,
        #[account(signer)]
        authority: AccountWithMetadata,
    ) -> LezResult {
        let state = CounterState {
            count: 0,
            owner: authority.account.id,
        };

        let mut updated = counter.account.clone();
        updated.data = state.to_bytes().try_into().unwrap();

        Ok(LezOutput::states_only(vec![
            AccountPostState::new_claimed(updated),
            AccountPostState::new(authority.account.clone()),
        ]))
    }

    /// Increment the counter. Only the owner may call this.
    #[instruction]
    pub fn increment(
        #[account(mut)]
        counter: AccountWithMetadata,
        #[account(signer)]
        authority: AccountWithMetadata,
    ) -> LezResult {
        let state = CounterState::from_bytes(&counter.account.data)
            .ok_or(LezError::DeserializationError { account_index: 0, message: "bad counter data".into() })?;

        if state.owner != authority.account.id {
            return Err(LezError::Unauthorized);
        }

        let new_state = CounterState { count: state.count + 1, owner: state.owner };
        let mut updated = counter.account.clone();
        updated.data = new_state.to_bytes().try_into().unwrap();

        Ok(LezOutput::states_only(vec![
            AccountPostState::new(updated),
            AccountPostState::new(authority.account.clone()),
        ]))
    }

    /// Read the current count. No state is modified.
    #[instruction]
    pub fn get_count(
        counter: AccountWithMetadata,
    ) -> LezResult {
        Ok(LezOutput::states_only(vec![
            AccountPostState::new(counter.account.clone()),
        ]))
    }
}
```

### Key concepts

**`AccountWithMetadata` has no generic type parameter.** Unlike some frameworks, there is no `AccountWithMetadata<CounterState>`. It is always just `AccountWithMetadata`. Account data is raw bytes accessed via `account.account.data`, deserialized manually.

**`LezResult`** is the return type alias for `Result<LezOutput, LezError>`. Always use `LezResult`, not the expanded form.

**`#[lez_program]`** marks the module as a LEZ program. The SPEL macro framework uses this to generate the dispatch table, IDL metadata, and guest wiring.

**`#[instruction]`** marks each public function as a callable instruction — analogous to a Solidity `external` function or a Solana instruction handler.

**Account parameters come first.** In every instruction, account arguments are declared before any data arguments. This is a hard requirement of the SPEL ABI; violating it causes a compile error.

**Account attributes:**

| Attribute | Meaning |
|---|---|
| `#[account(init)]` | The account is being created by this instruction |
| `#[account(mut)]` | The account's state may change |
| `#[account(signer)]` | The sequencer verifies a valid signature from this account's keypair |
| *(none)* | Read-only; the account's state is passed through unchanged |

**`AccountPostState`** — every account passed into an instruction must appear in the returned `post_states` vector. The sequencer uses this to verify that the instruction's state transition is complete and consistent.

- `AccountPostState::new_claimed(account)` — use this for accounts created by `#[account(init)]`. Takes the inner `Account` struct (from `account_with_metadata.account`), not the `AccountWithMetadata` wrapper.
- `AccountPostState::new(account)` — use this for all other accounts. Same: takes the inner `Account`.

> **⚠️ Warning:** If you forget to include an account in the `post_states` return value, the sequencer will reject the transaction with a state completeness error. Every account that enters an instruction must exit it.

---

## Step 4: Build

```bash
make build
# Equivalent manual command:
cargo build --release --jobs 2
```

The build compiles two targets:

1. The native host binary (used by `lez-cli` and the CLI wrapper)
2. The `riscv32im` guest binary that runs inside the RISC Zero zkVM to generate proofs

> **💡 Tip:** The first build takes significantly longer than subsequent builds because the RISC Zero guest compiler bootstraps from scratch. Expect several minutes on the first run. Subsequent incremental builds are much faster.

---

## Step 5: Generate the IDL

```bash
make idl
# Equivalent manual command:
cargo run --example generate_idl --release
```

This produces a `counter.json` IDL file (Interface Description Language — analogous to a Solidity ABI). A snippet of the generated output looks like:

```json
{
  "name": "counter",
  "version": "0.1.0",
  "instructions": [
    {
      "name": "initialize",
      "accounts": [
        { "name": "counter", "isMut": true, "isSigner": false, "isInit": true },
        { "name": "authority", "isMut": false, "isSigner": true }
      ],
      "args": []
    },
    {
      "name": "increment",
      "accounts": [
        { "name": "counter", "isMut": true, "isSigner": false },
        { "name": "authority", "isMut": false, "isSigner": true }
      ],
      "args": []
    },
    {
      "name": "get_count",
      "accounts": [
        { "name": "counter", "isMut": false, "isSigner": false }
      ],
      "args": []
    }
  ],
  "state": {
    "name": "CounterState",
    "fields": [
      { "name": "count", "type": "u64" },
      { "name": "owner", "type": { "array": ["u8", 32] } }
    ]
  }
}
```

> **⚠️ Warning:** Never edit `counter.json` by hand. It is a generated artifact. Any manual edits will be overwritten the next time you run `make idl`, and inconsistencies between the IDL and the compiled binary will cause runtime errors that are difficult to trace.

---

## Step 6: Start the Sequencer and Deploy

Open a second terminal for the sequencer if it isn't already running:

```bash
# Terminal 1 — keep running
cd ~/lez
RUST_LOG=info RISC0_DEV_MODE=1 ./target/release/sequencer_service --config sequencer/service/configs/debug/sequencer_config.json &
```

In your project terminal:

```bash
# Terminal 2 — deploy the program
make setup    # registers the program's ImageID with the sequencer
make deploy   # uploads the program binary
```

`make setup` submits a registration transaction that tells the sequencer: "the program with this ImageID is authorized to process state transitions." `make deploy` sends the compiled zkVM binary so the sequencer can verify proofs against it.

> **🔄 Coming from Solidity?** This two-step process maps roughly to deploying a contract with `CREATE` and then calling an initializer. The ImageID is the LEZ equivalent of a contract address — it uniquely identifies a specific program version. Unlike Ethereum addresses (which are random), the ImageID is deterministically derived from the compiled binary, so the same source code always produces the same ImageID.

---

## Step 7: Initialize a Counter Account

Every counter instance is a separate state account on-chain. Create one by calling the `initialize` instruction:

```bash
make cli ARGS="initialize --counter <account-address> --authority <your-genesis-account>"
```

Or call `lez-cli` directly:

```bash
lez-cli call --idl counter.json \
  --instruction initialize \
  --counter <account-address> \
  --authority <your-genesis-account>
```

Replace `<account-address>` with the address you want to use for the counter state account (this can be any valid address that doesn't already have state), and `<your-genesis-account>` with the address from `wallet account ls`.

---

## Step 8: Increment and Verify

```bash
# Send an increment transaction
make cli ARGS="increment --counter <account-address> --authority <your-genesis-account>"

# Inspect the raw account data
wallet account inspect <account-address>
```

The `wallet account inspect` output shows the raw bytes of the account's `data` field. You'll need to deserialize them manually using your `CounterState::from_bytes` function. After one `increment` call, the first 8 bytes should decode to `count: 1`.

> **🔄 Coming from Solidity?** In Solidity, you'd call `counter.getCount()` via `eth_call` — a read-only simulation that runs your contract code. **LEZ has no equivalent of `eth_call` or `view` functions.** There is no separate "read" path. State reads work by fetching raw account data from the sequencer and deserializing it off-chain. Use `wallet account inspect <account-id>` to view raw bytes, or `lez-cli --dry-run` to simulate an instruction call without submitting a transaction. The `get_count` instruction in our program exists only when you need a verified, ZK-proved read that appears in the transaction log.

---

## Reading State After a Transaction

This is one of the most common points of confusion for Solidity developers. There is no `view` function mechanism in LEZ. Here are your options:

**Option 1: Inspect raw account data**

```bash
wallet account inspect <account-id>
```

This shows the raw bytes stored in the account's `data` field. Deserialize them using your `CounterState::from_bytes` function off-chain.

**Option 2: Dry-run an instruction**

```bash
lez-cli call --idl counter.json \
  --instruction get_count \
  --dry-run \
  --counter <account-address>
```

`--dry-run` simulates the instruction without submitting it to the sequencer. The output shows what the account state would look like after the call.

**Option 3: Off-chain deserialization**

Fetch the raw account bytes from the sequencer RPC and deserialize in your client code. This is what your TypeScript or Rust client does when it reads state programmatically.

---

## What's Next

You now have a working LEZ program with:

- A state type serialized with Borsh
- Three instructions using SPEL macros
- A deployed program and an initialized state account
- A live increment transaction verified by the sequencer

The next chapters cover:

- **Multiple accounts per instruction** — programs that read and write several accounts in a single atomic transaction
- **Program-derived addresses (PDAs)** — deterministic account addresses owned by your program
- **Cross-program invocations** — calling another program's instructions from within yours
- **Client code generation** — using `lez-client-gen` to produce a typed TypeScript client from your IDL

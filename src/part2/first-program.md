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
├── Cargo.toml          # workspace manifest
├── Makefile            # build/deploy/call shortcuts
├── core/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs      # your program logic — SPEL macros live here
├── methods/
│   ├── Cargo.toml
│   └── guest/
│       └── src/
│           └── main.rs # zkVM guest entry point — wires core logic into the ZK proof
├── idl-gen/
│   └── src/
│       └── main.rs     # binary that generates the program's IDL JSON
└── cli-wrapper/
    └── src/
        └── main.rs     # CLI wrapper that lez-cli uses to call instructions
```

**Component roles:**

- **`core/`** — where you write your program. Contains your state types and instruction logic using SPEL macros. This is the only crate you'll edit for most programs.
- **`methods/`** — the RISC Zero zkVM guest crate. It imports `core` and exposes it to the zkVM prover. You generally don't edit this unless you need custom guest behavior.
- **`idl-gen/`** — a small binary that introspects your program and emits an IDL JSON file (like an ABI). Run it after changes to `core`.
- **`cli-wrapper/`** — auto-generated CLI glue that `lez-cli` uses to dispatch instruction calls. Regenerated from the IDL.

> **⚠️ Warning:** The scaffold from `lez-cli init` may be missing `risc0-zkvm` metadata in `methods/Cargo.toml`. If you get build errors about missing zkVM crate features or a missing `risc0` dependency, open `methods/Cargo.toml` and verify that the risc0 dependencies are present and have the correct feature flags for your toolchain version.

---

## Step 2: Define the State

Open `core/src/lib.rs` and define the state type for a counter account:

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use spel::prelude::*;

#[derive(BorshSerialize, BorshDeserialize, Default, Clone)]
pub struct CounterState {
    pub count: u64,
    pub owner: [u8; 32],
}
```

`CounterState` is serialized to and from account data using Borsh. Every state type in LEZ must implement `BorshSerialize` and `BorshDeserialize`.

> **⚠️ Warning:** `borsh_derive` does **not** compile for the `riscv32im` guest target. Do not add `borsh-derive` as a separate dependency. Instead, enable Borsh's built-in derive feature directly:
>
> ```toml
> # In core/Cargo.toml
> borsh = { version = "1", features = ["derive"] }
> ```
>
> Using `borsh_derive` as a standalone crate is one of the most common compilation errors for new LEZ developers. The derive macros in `borsh_derive` depend on `proc-macro2` internals that don't build cleanly for the RISC Zero guest target.

---

## Step 3: Write the Program Logic

Below `CounterState`, still in `core/src/lib.rs`, define the program module with its three instructions:

```rust
use spel::prelude::*;

#[lez_program]
mod counter {
    use super::*;

    /// Initialize a new counter account owned by `authority`.
    #[instruction]
    pub fn initialize(
        #[account(init)] counter: AccountWithMetadata<CounterState>,
        #[account(signer)] authority: AccountWithMetadata<()>,
    ) -> Result<LezOutput, LezError> {
        let mut state = counter.data().clone();
        state.owner = *authority.id();
        state.count = 0;

        let updated = counter.with_data(state);
        Ok(LezOutput::states_only(vec![
            AccountPostState::new_claimed(updated),
            AccountPostState::new(authority),
        ]))
    }

    /// Increment the counter. Only the owner may call this.
    #[instruction]
    pub fn increment(
        #[account(mut)] counter: AccountWithMetadata<CounterState>,
        #[account(signer)] authority: AccountWithMetadata<()>,
    ) -> Result<LezOutput, LezError> {
        let state = counter.data();
        if state.owner != *authority.id() {
            return Err(LezError::InvalidSigner);
        }

        let mut new_state = state.clone();
        new_state.count += 1;

        let updated = counter.with_data(new_state);
        Ok(LezOutput::states_only(vec![
            AccountPostState::new(updated),
            AccountPostState::new(authority),
        ]))
    }

    /// Read the current count. No state is modified.
    #[instruction]
    pub fn get_count(
        counter: AccountWithMetadata<CounterState>,
    ) -> Result<LezOutput, LezError> {
        Ok(LezOutput::states_only(vec![
            AccountPostState::new(counter),
        ]))
    }
}
```

### Key concepts

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

- `AccountPostState::new_claimed(account)` — use this for accounts created by `#[account(init)]`. It marks the account as now existing on-chain.
- `AccountPostState::new(account)` — use this for all other accounts (mutated or read-only pass-through).

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
cargo run --release -p idl-gen
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
sequencer_runner --port 3040
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

Replace `<account-address>` with the address you want to use for the counter state account (this can be any valid address that doesn't already have state), and `<your-genesis-account>` with the address from `wallet account list`.

---

## Step 8: Increment and Verify

```bash
# Send an increment transaction
make cli ARGS="increment --counter <account-address> --authority <your-genesis-account>"

# Read the account state directly from the sequencer
lez-cli account get <account-address>
```

The `account get` output will show the deserialized `CounterState` with the updated `count` field. After one `increment` call, you should see `count: 1`.

> **🔄 Coming from Solidity?** In Solidity, you'd call `counter.getCount()` to read state via an `eth_call`. In LEZ, `lez-cli account get` reads the account's raw state from the sequencer and deserializes it. There's no equivalent of a "view function" RPC call — state reads happen by fetching account data directly, not by executing program logic. The `get_count` instruction in our program exists for cases where you want a verified, ZK-proved read that appears in a transaction log.

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

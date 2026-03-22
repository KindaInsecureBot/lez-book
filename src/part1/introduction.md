# What is LEZ?

LEZ is a privacy-native smart contract platform where every program execution generates a zero-knowledge proof. It gives you the programmability of a smart contract system and the privacy of a ZK circuit — without requiring you to write a ZK circuit.

**LEZ** stands for **Logos Execution Zone** — one instantiation of the execution layer deployed for the **Logos Network** (logos.co), a decentralized governance and infrastructure stack built by IFT (Institute of Free Technology).

The naming history helps make sense of the acronyms you'll encounter:
- The project started as **Nescience** (privacy-first execution)
- The architecture became **NSSA** — "Nescience State Separation Architecture", later rebranded as **LSSA** (Logos State Separation Architecture)
- The execution environment was named **LEE** (Logos Execution Environment)
- **LEZ** (Logos Execution Zone) is the specific zone instantiation deployed by Logos

In practice, the terms you'll see daily:
- **LEZ** — the smart contract execution zone you're building for
- **NSSA/LSSA** — the underlying sequencer architecture (the `lssa` binary still uses this name)
- **LEE** — the execution environment spec that LEZ implements
- **SPEL** — the Rust macro framework for writing LEZ programs (github.com/logos-co/spel)
- **lssa** — the sequencer + wallet software (github.com/logos-blockchain/lssa)

> **💡 Tip:** If you're evaluating LEZ for a project, the mental model chapter is the fastest way to understand where it fits relative to tools you already know.

---

## The Core Idea

Programs on LEZ run inside the **RISC Zero zkVM**. When a user calls your program, the zkVM executes it and produces a cryptographic proof that the execution was correct. The sequencer (`lssa`) validates that proof and updates state — it never re-executes your code itself.

This is different from an EVM node, which re-executes every transaction on every validator. In LEZ, execution happens once, off-chain, and the proof travels to the sequencer. The sequencer only has to verify the proof, not redo the computation.

The result: computation scales down, privacy becomes possible, and correctness is cryptographically guaranteed.

---

## Privacy as a First-Class Feature

In most smart contract systems, privacy is an afterthought. EVM gives you programmability with zero privacy. ZK rollups add validity proofs but the privacy question is left to application-layer hacks:

- **Tornado Cash**: a mixer bolted on top of Ethereum
- **Aztec**: a separate chain with its own circuit DSL (Noir)
- **Zcash shielded pools**: purpose-built, not general-purpose

LEZ takes a different approach: **the account model itself is privacy-aware from day one**.

Every LEZ program natively supports both:

- **Public transactions** — inputs and outputs visible on-chain, just like normal contract calls
- **Private transactions** — inputs and outputs hidden using commitments and nullifiers (Zcash-style), but for arbitrary program state

The key point is that these two modes use **the same compiled program binary**. You don't write a public version and a private version. You write one program. The privacy circuit wraps it at the protocol level, applied by `lssa` when a private transaction is submitted.

You write normal Rust. The ZK machinery is underneath you, not in your face.

---

## Architecture Overview

LEZ has three major components:

### `lssa` — The Sequencer

`lssa` is the sequencer and wallet daemon. It replaces what a blockchain node does in EVM land, but with a fundamentally different trust model:

- Receives transaction submissions (as ZK proof + public inputs)
- Verifies the RISC Zero proof
- Updates the global account state
- Manages the canonical ledger

There is no block production in the traditional sense. `lssa` is the single sequencer that processes transactions in order. It does not re-execute programs — it only verifies their proofs.

### SPEL — The Macro Framework

SPEL is the Rust macro layer that turns ordinary Rust code into a LEZ program. It handles:

- Serialization/deserialization of accounts and instructions (via Borsh)
- Account constraint checking (`signer`, `owner`, `init`, etc.)
- IDL (Interface Definition Language) generation for client tooling
- Program dispatch — routing instruction calls to the right handler

The two main macros you'll use constantly are `#[lez_program]` and `#[instruction]`.

### `lez-cli` — Developer Tooling

`lez-cli` is the command-line interface for building, testing, and deploying LEZ programs. It wraps the RISC Zero toolchain and `lssa` RPC calls into ergonomic commands:

```bash
lez-cli init my-program     # Scaffold a new program workspace
lez-cli build               # Compile to riscv32im and generate ImageID
lez-cli deploy              # Deploy to lssa and register the ImageID
lez-cli call <instruction>  # Call an instruction on a deployed program
```

The scaffolded workspace includes a `Makefile` with `make deploy` and `make cli ARGS="..."` targets so you rarely need to type full `lez-cli` invocations directly.

---

## State Model: Accounts, Not Storage Slots

If you're coming from Solidity, the biggest conceptual shift is where state lives.

In Solidity, state lives *inside* the contract — in storage slots keyed by the contract's address. Your contract is both the code and the database.

In LEZ, state lives in **accounts** — separate objects that exist outside the program. Every instruction receives the accounts it needs as explicit inputs. The program reads them, mutates them, and returns updated versions. The program itself is stateless.

This is closer to the Solana account model than Ethereum's. If you've used Anchor (Solana's Rust framework), the pattern will feel familiar.

An account has:
- A unique address (either user-chosen or PDA-derived)
- A balance (in the native token)
- A nonce (for replay protection)
- An owner (the ImageID of the program allowed to write to it)
- Data (a Borsh-serialized struct)

---

## Program Identity: ImageID

On Ethereum, a contract's address identifies it. On LEZ, a program is identified by its **RISC Zero ImageID** — the hash of the compiled zkVM guest binary.

This has a direct implication for upgrades: if you change your code, the ImageID changes, and that is effectively a new program. Accounts owned by the old ImageID cannot be written by the new one. Upgradeability requires deliberate design (migration instructions, a governance layer, or an explicit upgrade mechanism).

This is a security feature, not an oversight. It means that the code that executed a transaction is cryptographically committed in the proof — there's no way for an owner to silently swap the implementation.

---

## Writing a LEZ Program

Programs are written in Rust using the SPEL macro framework. A minimal program looks like this:

```rust
#![no_main]
use nssa_core::account::AccountWithMetadata;
use nssa_core::program::AccountPostState;
use lez_framework::prelude::*;
risc0_zkvm::guest::entry!(main);

#[lez_program]
mod counter {
    #[allow(unused_imports)]
    use super::*;

    #[instruction]
    pub fn increment(
        #[account(mut, pda = literal("counter"))]
        counter: AccountWithMetadata,
        #[account(signer)]
        authority: AccountWithMetadata,
    ) -> LezResult {
        let current = u64::from_le_bytes(counter.account.data[..8].try_into().unwrap());
        let new_value = current + 1;
        let mut updated = counter.account.clone();
        updated.data = new_value.to_le_bytes().to_vec().try_into().unwrap();
        Ok(LezOutput::states_only(vec![
            AccountPostState::new(updated),
            AccountPostState::new(authority.account.clone()),
        ]))
    }
}
```

Key differences from other account-based chains:
- **`AccountWithMetadata` has no generic type parameter** — account data is raw bytes (`account.data`), deserialized manually
- **`#![no_main]` and `risc0_zkvm::guest::entry!(main)`** are required for the RISC Zero guest binary
- **`LezResult`** is the return type alias for `Result<LezOutput, LezError>`
- **No `require!` macro** — use standard `if / return Err(...)` patterns
- Account data is accessed as `counter.account.data` (a `Vec<u8>`) and deserialized manually

The `#[lez_program]` macro wraps the module and generates the program entry point. The `#[instruction]` macro generates dispatch logic and IDL entries for each callable function. Account constraints like `#[account(signer)]` and `#[account(mut)]` are checked before your function body runs.

---

## The Developer Workflow

End-to-end, building and deploying a LEZ program looks like this:

```bash
# 1. Scaffold
lez-cli init my-counter
cd my-counter

# 2. Write your program in src/lib.rs

# 3. Build (compiles to riscv32im, derives ImageID)
make build

# 4. Start lssa locally
lssa --dev &

# 5. Deploy (registers ImageID, creates program account)
make deploy

# 6. Initialize program state
make cli ARGS="initialize --owner <pubkey>"

# 7. Call instructions
make cli ARGS="increment --counter <account-address>"
```

The same `make cli` command works for both public and private transaction modes — you add a `--private` flag for privacy-preserving execution.

---

## Summary

| Concept | LEZ |
|---|---|
| Language | Rust + SPEL macros |
| Execution | RISC Zero zkVM (generates ZK proof) |
| State | External accounts (not inside the program) |
| Program identity | RISC Zero ImageID |
| Privacy | Native — same binary handles public and private |
| Sequencer | `lssa` (validates proofs, updates state) |
| Developer CLI | `lez-cli` |

LEZ is not a rollup on top of Ethereum. It is not an EVM chain with a ZK prover stapled on. It is a purpose-built platform where ZK proofs and account-based privacy are architectural primitives, not features added later.

The next chapter maps the concepts you know from Solidity and the EVM to their LEZ equivalents — concretely, with code examples.

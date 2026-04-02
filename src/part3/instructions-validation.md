# Instructions & Validation

## The #[lez_program] Macro

Every LEZ program starts with:

```rust
#[lez_program]
mod my_program {
    // instructions here
}
```

This macro generates the zkVM guest entry point, the dispatch table for instructions, and the IDL metadata.

## The #[instruction] Macro

Each callable function gets `#[instruction]`:

```rust
#[instruction]
pub fn my_instruction(
    // accounts first, then arguments
    #[account(signer)] caller: AccountWithMetadata,
    #[account(mut)] target: AccountWithMetadata,
    value: u64,
) -> LezResult {
    // ...
}
```

## Account Constraints

The full set of constraints:

```rust
#[account(signer)]                                      // account must sign the transaction
#[account(init)]                                        // account must be default/empty — implies mut
#[account(mut)]                                         // account is writable
#[account(pda = literal("x"))]                         // verify PDA from constant seed
#[account(pda = account("u"))]                         // verify PDA from another account's ID
#[account(pda = arg("key"))]                           // verify PDA from instruction argument
#[account(pda = [literal("v"), account("u")])]         // multi-seed PDA
#[account(owner = PROGRAM_ID)]                         // account must be owned by this program
```

Constraints can be combined (except where noted):

```rust
// init + pda — new account at a derived address
#[account(init, pda = [literal("player"), account("user")])]

// mut + pda — existing writable PDA
#[account(mut, pda = arg("key"))]

// mut + owner — writable account owned by this program
#[account(mut, owner = MY_PROGRAM_ID)]
```

> **⚠️ Warning:** `init` implies `mut`. Don't combine them (`#[account(init, mut)]` is redundant and may cause issues). Use just `#[account(init)]`.

## Parameter Ordering Rule

**Accounts MUST come before arguments** in instruction function signatures. This is enforced by the macro system.

```rust
// ✅ Correct — accounts first, then scalar args
#[instruction]
pub fn claim(
    #[account(init, pda = [literal("tile"), arg("x"), arg("y")])] tile: AccountWithMetadata,
    #[account(signer)] player: AccountWithMetadata,
    x: u64, // args after accounts
    y: u64,
) -> LezResult { ... }

// ❌ Wrong — arg before account
#[instruction]
pub fn claim(
    x: u64,
    #[account(init)] tile: AccountWithMetadata, // COMPILE ERROR
    #[account(signer)] player: AccountWithMetadata,
) -> LezResult { ... }
```

## Return Type

All instructions return `LezResult` (a type alias for `Result<LezOutput, LezError>`).

The most common return patterns:

```rust
// States only (no CPI)
Ok(LezOutput::states_only(vec![
    AccountPostState::new_claimed(new_account),  // for init accounts
    AccountPostState::new(existing_account),      // for updated accounts
]))

// With chained calls (CPI)
Ok(LezOutput::with_chained_calls(
    vec![AccountPostState::new(account)],
    vec![chained_call],
))
```

## Error Handling

Return errors with `LezError` variants:

```rust
if caller.account.id != state.owner {
    return Err(LezError::Unauthorized);
}

if state.count >= MAX_COUNT {
    return Err(LezError::Custom(6001)); // custom error code (use 6000+)
}
```

Real `LezError` variants:

| Variant | Code | Meaning |
|---|---|---|
| `LezError::AccountCountMismatch` | 1000 | Wrong number of accounts passed |
| `LezError::InvalidAccountOwner` | 1001 | Account not owned by expected program |
| `LezError::AccountAlreadyInitialized` | 1002 | `#[account(init)]` account already has data |
| `LezError::Unauthorized` | 1008 | Authorization check failed (use instead of "InvalidSigner") |
| `LezError::SerializationError { message }` | — | Failed to serialize account data |
| `LezError::DeserializationError { account_index, message }` | — | Failed to deserialize account data |
| `LezError::Custom(u32)` | 6000+ | User-defined error code (use 6000+ range) |

> **⚠️ Warning:** `LezError::InvalidSigner`, `LezError::AccountNotInit`, `LezError::AccountAlreadyInit`, and `LezError::InvalidOwner` do **not exist**. Using them will fail to compile. Use the variants listed above.

## Automatic vs Manual Validation

SPEL handles some checks automatically via the macro:

- `#[account(signer)]` — verified by the protocol before your code runs
- `#[account(init)]` — account must be empty/default (protocol check)
- `#[account(pda = ...)]` — address verified against derivation

Manual checks you still need to do:

- Business logic (ownership, value ranges, etc.)
- Custom authorization (e.g., "caller must be stored owner")

> **🔄 Coming from Solidity?** Solidity modifiers (`onlyOwner`, `nonReentrant`) are analogous to account constraints. But where Solidity modifiers are runtime checks defined in your code, SPEL constraints are checked at the protocol level before your instruction code even runs.

## A Complete Example: Owner-Only Counter

```rust
#![no_main]
use nssa_core::account::AccountWithMetadata;
use nssa_core::program::AccountPostState;
use spel_framework::prelude::*;
risc0_zkvm::guest::entry!(main);

#[lez_program]
mod counter {
    #[allow(unused_imports)]
    use super::*;

    #[instruction]
    pub fn initialize(
        #[account(init)] counter: AccountWithMetadata,
        #[account(signer)] owner: AccountWithMetadata,
    ) -> LezResult {
        let state = CounterState {
            count: 0,
            owner: owner.account.id,
        };
        let mut new_counter = counter.account.clone();
        new_counter.data = state.to_bytes().try_into().unwrap();
        Ok(LezOutput::states_only(vec![
            AccountPostState::new_claimed(new_counter),
            AccountPostState::new(owner.account.clone()),
        ]))
    }

    #[instruction]
    pub fn increment(
        #[account(mut, owner = COUNTER_PROGRAM_ID)] counter: AccountWithMetadata,
        #[account(signer)] caller: AccountWithMetadata,
    ) -> LezResult {
        let state = CounterState::from_bytes(&counter.account.data)
            .ok_or(LezError::DeserializationError { account_index: 0, message: "bad counter data".into() })?;

        // Manual business logic check — the stored owner must match the caller
        if state.owner != caller.account.id {
            return Err(LezError::Unauthorized);
        }

        let new_state = CounterState { count: state.count + 1, owner: state.owner };
        let mut updated = counter.account.clone();
        updated.data = new_state.to_bytes().try_into().unwrap();

        Ok(LezOutput::states_only(vec![
            AccountPostState::new(updated),
            AccountPostState::new(caller.account.clone()),
        ]))
    }
}
```

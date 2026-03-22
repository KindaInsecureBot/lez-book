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
    #[account(signer)] caller: AccountWithMetadata<()>,
    #[account(mut)] target: AccountWithMetadata<MyState>,
    value: u64,
) -> Result<LezOutput, LezError> {
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
    #[account(init, pda = [literal("tile"), arg("x"), arg("y")])] tile: AccountWithMetadata<TileState>,
    #[account(signer)] player: AccountWithMetadata<()>,
    x: u64, // args after accounts
    y: u64,
) -> Result<LezOutput, LezError> { ... }

// ❌ Wrong — arg before account
#[instruction]
pub fn claim(
    x: u64,
    #[account(init)] tile: AccountWithMetadata<TileState>, // COMPILE ERROR
    #[account(signer)] player: AccountWithMetadata<()>,
) -> Result<LezOutput, LezError> { ... }
```

## Return Type

All instructions return `Result<LezOutput, LezError>`.

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
if caller.id() != &state.owner {
    return Err(LezError::InvalidSigner);
}

if state.count >= MAX_COUNT {
    return Err(LezError::Custom(42)); // custom error code
}
```

Common `LezError` variants:

| Variant | Meaning |
|---|---|
| `LezError::InvalidSigner` | Signer check failed |
| `LezError::AccountNotInit` | Expected init account that already exists |
| `LezError::AccountAlreadyInit` | Expected existing account that's empty |
| `LezError::InvalidOwner` | Owner check failed |
| `LezError::Custom(u32)` | Custom error code for your program |

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
#[lez_program]
mod counter {
    use super::*;

    #[instruction]
    pub fn initialize(
        #[account(init)] counter: AccountWithMetadata<CounterState>,
        #[account(signer)] owner: AccountWithMetadata<()>,
    ) -> Result<LezOutput, LezError> {
        let state = CounterState {
            count: 0,
            owner: *owner.id(),
        };
        Ok(LezOutput::states_only(vec![
            AccountPostState::new_claimed(counter.with_data(state)),
            AccountPostState::new(owner),
        ]))
    }

    #[instruction]
    pub fn increment(
        #[account(mut, owner = COUNTER_PROGRAM_ID)] counter: AccountWithMetadata<CounterState>,
        #[account(signer)] caller: AccountWithMetadata<()>,
    ) -> Result<LezOutput, LezError> {
        // Manual business logic check
        if &counter.data().owner != caller.id() {
            return Err(LezError::InvalidSigner);
        }
        let mut state = counter.data().clone();
        state.count += 1;
        Ok(LezOutput::states_only(vec![
            AccountPostState::new(counter.with_data(state)),
            AccountPostState::new(caller),
        ]))
    }
}
```

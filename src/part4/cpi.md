# Cross-Program Invocation

## What Is CPI?

Cross-Program Invocation (CPI) lets a program call instructions on other programs within the same transaction. The calling program's output includes a list of "chained calls" that the sequencer executes next.

> **🔄 Coming from Solidity?** CPI is conceptually similar to `external contract calls` (not `delegatecall` — your code doesn't run in a foreign context). Like Solidity external calls, CPI passes execution to another program, which has its own logic and state. Unlike Solidity, there's no reentrancy risk because execution is sequential and each step is proven independently.

## Execution Model

When your program returns a `ChainedCall`, the sequencer:

1. Completes your program's ZK proof
2. Applies your program's state updates
3. Executes the chained call with the updated state visible
4. Generates a separate ZK proof for the chained call
5. Continues until no more chained calls remain

Key properties:
- **Sequential**: chained calls execute one at a time, in order
- **Each gets its own ZK proof**: the sequencer proves each step independently
- **State propagates**: account state from step N is visible to step N+1 — if your program modifies an account and then chains a call that reads it, the chained call sees the updated state
- **The calling program must know the target program ID** at the time it returns the `ChainedCall` (either hardcoded or passed as an argument)

## The ChainedCall Structure

```rust
use spel::prelude::*;

// Build a chained call to another program
let call = ChainedCall {
    program_id: OTHER_PROGRAM_ID,  // the program to call
    instruction_name: "deposit".to_string(),
    accounts: vec![
        CallAccount { id: vault_account.id().clone(), writable: true, signer: false },
        CallAccount { id: caller.id().clone(), writable: false, signer: true },
    ],
    args: borsh::to_vec(&DepositArgs { amount: 100 })?,
};
```

## Returning with Chained Calls

Use `LezOutput::with_chained_calls` instead of `states_only`:

```rust
#[instruction]
pub fn swap(
    #[account(mut)] pool: AccountWithMetadata<PoolState>,
    #[account(signer)] user: AccountWithMetadata<()>,
    amount_in: u64,
) -> Result<LezOutput, LezError> {
    // Update pool state
    let new_pool_state = PoolState {
        reserve_a: pool.data().reserve_a + amount_in,
        reserve_b: pool.data().reserve_b - compute_out(pool.data(), amount_in),
    };
    let updated_pool = pool.with_data(new_pool_state);

    // Build chained call to token program for the actual token transfer
    let transfer_call = ChainedCall {
        program_id: TOKEN_PROGRAM_ID,
        instruction_name: "transfer".to_string(),
        accounts: vec![
            CallAccount { id: *user.id(), writable: true, signer: true },
            CallAccount { id: pool_token_account_id, writable: true, signer: false },
        ],
        args: borsh::to_vec(&TransferArgs { amount: amount_in })?,
    };

    Ok(LezOutput::with_chained_calls(
        vec![
            AccountPostState::new(updated_pool),
            AccountPostState::new(user),
        ],
        vec![transfer_call],
    ))
}
```

The swap program updates the pool reserve ratios, then chains a call to the token program to actually move the tokens. The token program runs next, sees the already-updated pool state, and processes the transfer.

## Passing Binaries with --bin-\<name\>

When calling a program via CPI, the sequencer needs the binary of the called program to execute and prove it. The `--bin-<name>` flag provides the guest binary and automatically fills in the corresponding `--<name>-program-id`:

```bash
lez-cli call --idl swap.json \
  --instruction swap \
  --bin-token-program ./target/riscv-guest/release/token_program \
  --pool <pool-addr> \
  --user <user-addr> \
  --amount-in 100
```

The `--bin-token-program` flag does two things:
1. Sends the `token_program` binary to the sequencer
2. Derives and fills in `--token-program-id` (the ImageID of that binary)

Without the binary, the sequencer cannot execute the chained call and the transaction fails.

## Real-World Pattern: Swap → Token Transfer

A common CPI pattern is a DEX swap calling into a token program:

```rust
// In the swap program: TOKEN_PROGRAM_ID is a compile-time constant
// (or could be passed as a verified argument)
const TOKEN_PROGRAM_ID: [u32; 8] = [/* risc zero image id of token program */];

#[instruction]
pub fn swap(
    #[account(mut)] pool: AccountWithMetadata<PoolState>,
    #[account(mut)] user_token_in: AccountWithMetadata<TokenAccount>,
    #[account(mut)] pool_token_in: AccountWithMetadata<TokenAccount>,
    #[account(mut)] pool_token_out: AccountWithMetadata<TokenAccount>,
    #[account(mut)] user_token_out: AccountWithMetadata<TokenAccount>,
    #[account(signer)] user: AccountWithMetadata<()>,
    amount_in: u64,
) -> Result<LezOutput, LezError> {
    let amount_out = compute_out(pool.data(), amount_in);

    // Update pool state (AMM invariant)
    let updated_pool = pool.with_data(/* new reserves */);

    // Chained call 1: transfer tokens IN (user → pool)
    let transfer_in = ChainedCall {
        program_id: TOKEN_PROGRAM_ID,
        instruction_name: "transfer".to_string(),
        accounts: vec![
            CallAccount { id: *user_token_in.id(), writable: true, signer: false },
            CallAccount { id: *pool_token_in.id(), writable: true, signer: false },
            CallAccount { id: *user.id(), writable: false, signer: true },
        ],
        args: borsh::to_vec(&TransferArgs { amount: amount_in })?,
    };

    // Chained call 2: transfer tokens OUT (pool → user)
    let transfer_out = ChainedCall {
        program_id: TOKEN_PROGRAM_ID,
        instruction_name: "transfer".to_string(),
        accounts: vec![
            CallAccount { id: *pool_token_out.id(), writable: true, signer: false },
            CallAccount { id: *user_token_out.id(), writable: true, signer: false },
            CallAccount { id: *pool.id(), writable: false, signer: true },
        ],
        args: borsh::to_vec(&TransferArgs { amount: amount_out })?,
    };

    Ok(LezOutput::with_chained_calls(
        vec![
            AccountPostState::new(updated_pool),
            AccountPostState::new(user_token_in),
            AccountPostState::new(pool_token_in),
            AccountPostState::new(pool_token_out),
            AccountPostState::new(user_token_out),
            AccountPostState::new(user),
        ],
        vec![transfer_in, transfer_out],
    ))
}
```

## CPI Constraints

- **Sequential**: chained calls execute in order, not in parallel
- **Target program ID required**: the calling program must know the called program's ID at the time it builds the `ChainedCall`. The ID can be hardcoded or passed as a verified argument — but the caller is responsible for verifying it's the right program
- **Each chained call generates its own ZK proof**: longer chains = more proving time
- **State propagates forward**: account updates from step N are visible to step N+1

> **⚠️ Warning:** There is no reentrancy guard because there's no reentrancy — execution is strictly sequential and each step is fully proven before the next begins. However, you should still be careful about the order of state updates vs. chained calls. Build your state updates first, then chain calls that depend on them.

> **💡 Tip:** Because the calling program must know the target program ID, CPI works best when the called program is a stable, well-known program (like a system token program) rather than a user-supplied address. If you need dynamic dispatch, pass the program ID as an argument and validate it against an allow-list in your program logic.

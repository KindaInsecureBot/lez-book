# Cross-Program Invocation

## What Is CPI?

Cross-Program Invocation (CPI) lets a program call instructions on other programs within the same transaction. The calling program's output includes a list of "chained calls" that the sequencer executes next.

> **🔄 Coming from Solidity?** CPI is conceptually similar to `external contract calls` (not `delegatecall` — your code doesn't run in a foreign context). Like Solidity external calls, CPI passes execution to another program, which has its own logic and state. Unlike Solidity, there's no reentrancy risk because execution is sequential within the ZK proof.

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
    // ... update pool state ...
    let updated_pool = pool.with_data(new_pool_state);

    // Build chained call to token program
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

## Passing Binaries with --bin-\<name\>

When deploying programs that use CPI, you need to pass the binary of the called program:

```bash
lez-cli call --idl swap.json \
  --instruction swap \
  --bin-token-program ./target/release/token_program \
  --pool <pool-addr> \
  --user <user-addr> \
  --amount-in 100
```

The `--bin-<name>` flag provides the guest binary of the called program so the sequencer can execute it.

## CPI Constraints

- Chained calls are executed sequentially
- The calling program must know the target program's ID at compile time (or pass it as an arg)
- Each chained call generates its own ZK proof
- Account state updates from the calling program are visible to the chained call

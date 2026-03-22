# Appendix C: Error Code Reference

This appendix documents the error types your program can return, how to define custom error codes, and the sequencer-level errors you may encounter.

---

## LezError Variants

`LezError` is the standard error type returned from instruction functions (`Result<LezOutput, LezError>`).

| Variant | When It's Returned |
|---|---|
| `InvalidSigner` | Signer check failed, or the wrong account attempted to sign |
| `AccountNotInit` | Expected an initialized account but found an empty/default one |
| `AccountAlreadyInit` | `#[account(init)]` was used on an account that already has data |
| `InvalidOwner` | `#[account(owner = ...)]` check failed — wrong program owns this account |
| `InvalidPDA` | PDA derivation of the passed account doesn't match the expected address |
| `InsufficientFunds` | Account balance too low for the operation |
| `Overflow` | Arithmetic overflow detected |
| `Custom(u32)` | Application-defined error code |

### Usage Example

```rust
use lez_sdk::LezError;

pub fn transfer(
    #[account(mut)] from: AccountWithMetadata<WalletState>,
    #[account(mut)] to: AccountWithMetadata<WalletState>,
    #[account(signer)] owner: AccountWithMetadata<()>,
    amount: u64,
) -> Result<LezOutput, LezError> {
    // Check authorization
    if &from.data().owner != owner.id() {
        return Err(LezError::InvalidSigner);
    }

    // Check balance
    if from.data().balance < amount {
        return Err(LezError::InsufficientFunds);
    }

    // Safe arithmetic
    let new_from_balance = from.data().balance
        .checked_sub(amount)
        .ok_or(LezError::Overflow)?;

    let new_to_balance = to.data().balance
        .checked_add(amount)
        .ok_or(LezError::Overflow)?;

    // ... rest of transfer
}
```

---

## Custom Error Codes

Define your program's custom error codes as named constants and return them via `LezError::Custom(code)`:

```rust
// In your core crate (visible to both guest and host)
pub const ERR_NOT_OWNER: u32 = 1;
pub const ERR_TILE_TAKEN: u32 = 2;
pub const ERR_NOT_ADJACENT: u32 = 3;
pub const ERR_INSUFFICIENT_TILES: u32 = 4;
pub const ERR_INVALID_COORDINATES: u32 = 5;
pub const ERR_ALREADY_CLAIMED: u32 = 6;

// Usage in an instruction
if tile.data().owner_commitment != [0u8; 32] {
    return Err(LezError::Custom(ERR_TILE_TAKEN));
}
```

> **💡 Tip:** Document your custom error codes in your program's IDL and README. Clients that receive a `Custom(2)` error have no way to know what it means unless it's documented.

### Recommended Convention

Start custom error codes at 1 (reserve 0 for "no error" in case you use it in non-Result contexts). Group related errors together and leave gaps for future additions:

```rust
// Account/ownership errors: 1-9
pub const ERR_NOT_OWNER: u32 = 1;
pub const ERR_WRONG_PROGRAM: u32 = 2;

// Game logic errors: 10-19
pub const ERR_TILE_TAKEN: u32 = 10;
pub const ERR_NOT_ADJACENT: u32 = 11;
pub const ERR_INVALID_COORDINATES: u32 = 12;

// Resource errors: 20-29
pub const ERR_INSUFFICIENT_TILES: u32 = 20;
pub const ERR_INSUFFICIENT_BALANCE: u32 = 21;
```

---

## Common Sequencer Errors

These errors originate in the sequencer (`lssa`), not your program. They appear in RPC responses when a transaction is rejected before or after proof verification.

| Error | Cause | Resolution |
|---|---|---|
| `ProofVerificationFailed` | The ZK proof didn't verify — binary mismatch or corruption | Rebuild the guest binary and redeploy; ensure `RISC0_DEV_MODE=0` |
| `InvalidNonce` | Transaction nonce is wrong — replay or out-of-order tx | Check the account's current nonce; use sequential nonces |
| `AccountNotFound` | Referenced account doesn't exist on chain | Create the account before referencing it, or use `init` |
| `ProgramNotFound` | Program ImageID not registered with the sequencer | Deploy the program with `lez-cli deploy` |
| `InvalidCommitment` | Commitment hash doesn't match (private transaction) | Re-derive the commitment; check your nsk/npk |
| `StaleProof` | Proof references outdated state root | Retry the transaction against the current state |

### ProofVerificationFailed in Detail

This is the most confusing sequencer error because it can have several causes:

1. **Binary mismatch**: The deployed ImageID doesn't match the binary you're using. Redeploy after rebuilding.
2. **RISC0_DEV_MODE**: You built in dev mode (fake proofs) but the sequencer requires real proofs. Rebuild with `RISC0_DEV_MODE=0`.
3. **Corrupted ELF**: OOM during build produced a corrupted binary. Rebuild with `--jobs 2`.
4. **Toolchain mismatch**: The RISC Zero toolchain version used to build doesn't match the verifier. Use `rzup` to ensure consistent versions.

---

## Error Handling in Clients

Generated client code wraps sequencer and program errors in a typed enum. When the sequencer returns an error, it includes the program's `Custom(u32)` code if applicable.

```rust
match client.call_instruction(...).await {
    Ok(receipt) => { /* success */ }
    Err(LezClientError::ProgramError(LezError::Custom(code))) => {
        match code {
            ERR_TILE_TAKEN => println!("That tile is already claimed"),
            ERR_NOT_ADJACENT => println!("You can only claim adjacent tiles"),
            _ => println!("Unknown program error: {}", code),
        }
    }
    Err(LezClientError::SequencerError(msg)) => {
        println!("Sequencer rejected transaction: {}", msg);
    }
    Err(e) => println!("Unexpected error: {}", e),
}
```

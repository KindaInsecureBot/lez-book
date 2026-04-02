# Appendix C: Error Code Reference

This appendix documents the `LezError` variants your program can return, the framework error codes the sequencer produces, and how to define and return custom error codes.

---

## LezError Variants

`LezError` is the error enum returned from all LEZ instruction functions. The return type of an instruction is `LezResult`, which is a type alias for `Result<LezOutput, LezError>`.

```rust
pub type LezResult = Result<LezOutput, LezError>;
```

---

## Framework Errors (Codes 1000–1999)

These errors are produced by the LEZ framework and sequencer. You do not define them — they appear when framework-level constraints are violated.

| Code | Variant | Meaning |
|------|---------|---------|
| 1000 | `AccountCountMismatch` | The number of accounts returned in `LezOutput` does not match the number of accounts passed in to the instruction. Every input account must be included in `post_states`. |
| 1001 | `InvalidAccountOwner` | An account's owner field does not match the expected program ID. This occurs when you pass an account owned by a different program where a specific program's account is expected. |
| 1002 | `AccountAlreadyInitialized` | An `#[account(init)]` constraint found that the account already contains data. You cannot re-initialize an existing account with `init`. |
| 1008 | `Unauthorized` | A signer check failed. The account marked `#[account(signer)]` did not produce a valid signature for this transaction. |

### What These Errors Mean in Practice

**`AccountCountMismatch` (1000)**

This is the most common framework error during development. It means you returned fewer accounts in `LezOutput::states_only(...)` than were passed in. Every account that the sequencer sends to your instruction — including ones you never read — must be returned.

```rust
// If 3 accounts come in, 3 must go out
Ok(LezOutput::states_only(vec![
    account_a.into_post_state(data_a),
    account_b.into_post_state(data_b),  // even if unchanged
    account_c.into_post_state(data_c),
]))
```

**`InvalidAccountOwner` (1001)**

Usually means the caller passed the wrong account, or the account was created by a different program version. This also occurs if you pass a system-owned account (not yet initialized) where a program-owned account is expected.

Common causes:
- Passing a genesis account where a PDA is expected
- Mixing accounts from different program versions (old ImageID vs. new ImageID)
- Passing accounts in the wrong order (the second account where the first is expected)

**`AccountAlreadyInitialized` (1002)**

Occurs when calling an instruction with `#[account(init)]` on an account that already has data. If you need to re-initialize or reset an account, write a separate `reset` or `close` instruction that explicitly clears the data, then call `initialize` again.

**`Unauthorized` (1008)**

A signature check failed. The most common cause is passing a PDA where a genesis (keypair) account is required. PDAs cannot produce signatures. See [Gotchas #1](../part5/gotchas.md#1-genesis-accounts-must-be-signers--not-pdas).

---

## Serialization Errors

These errors occur when borsh serialization or deserialization fails. They are not assigned numeric codes — they are enum variants with associated data.

| Variant | When It Occurs |
|---------|----------------|
| `SerializationError` | `BorshSerialize` failed when writing account data. Usually means a type's serialization implementation has a bug. |
| `DeserializationError` | `BorshDeserialize` failed when reading account data. The most common cause is reading an account with the wrong struct type. |

### What `DeserializationError` Usually Means

If you get a `DeserializationError`, the most common causes are:

1. **Wrong struct type:** You're deserializing an account into `StructA` but the account was initialized with `StructB`. Account types are not tagged — if you pass the wrong struct, deserialization silently reads garbage or fails when it runs out of bytes.

2. **Stale account data:** The account was created by an old version of the program with a different struct layout. Adding or reordering fields in a struct is a breaking change for existing accounts.

3. **Uninitialized account:** The account has never been written (it's a fresh, empty account) and you're trying to deserialize it. Check for `AccountAlreadyInitialized` or verify the account was initialized before reading.

4. **Off-by-one in manual borsh:** If you manually implement `BorshSerialize` and `BorshDeserialize`, a mismatch in field order between the two impls will cause deserialization to fail or return wrong values silently.

```rust
// Always deserialize defensively
let state = match MyState::try_from_slice(&account.data) {
    Ok(s) => s,
    Err(_) => return Err(LezError::DeserializationError),
};
```

---

## Custom User Errors (Codes 6000+)

Define your program's domain errors as `u32` constants starting at 6000. This avoids collision with framework error codes (1000–1999) and leaves room for future framework additions.

### Defining Custom Errors

```rust
// In your program's error constants file or at the top of lib.rs

// Game program errors
pub const ERR_INVALID_COORDINATES: u32 = 6000;
pub const ERR_TILE_OCCUPIED: u32       = 6001;
pub const ERR_INSUFFICIENT_FUNDS: u32  = 6002;
pub const ERR_GAME_ALREADY_OVER: u32   = 6003;
pub const ERR_NOT_YOUR_TURN: u32       = 6004;
pub const ERR_INVALID_MOVE: u32        = 6005;
```

### Returning Custom Errors

```rust
#[instruction]
pub fn place_tile(
    board: AccountWithMetadata,
    player: AccountWithMetadata,
    x: u32,
    y: u32,
) -> LezResult {
    let board_state = BoardState::try_from_slice(&board.data)
        .map_err(|_| LezError::DeserializationError)?;

    // Validate coordinates
    if x >= BOARD_SIZE || y >= BOARD_SIZE {
        return Err(LezError::Custom(ERR_INVALID_COORDINATES));
    }

    // Check if tile is occupied
    if board_state.tiles[x as usize][y as usize] != 0 {
        return Err(LezError::Custom(ERR_TILE_OCCUPIED));
    }

    // ... proceed with placement ...
}
```

### Numbering Convention

Start at 6000 and number sequentially. Use a comment block at the top of your error constants to document each code:

```rust
// ──────────────────────────────────────────────────
// Custom Error Codes
// Range: 6000–6999
// ──────────────────────────────────────────────────
// 6000: Coordinate out of board bounds
// 6001: Target tile already occupied by another player
// 6002: Player's token balance insufficient for this move
// 6003: Game session has already ended
// 6004: Action submitted out of turn order
// 6005: Move violates game rules
// ──────────────────────────────────────────────────
pub const ERR_INVALID_COORDINATES: u32  = 6000;
pub const ERR_TILE_OCCUPIED: u32        = 6001;
pub const ERR_INSUFFICIENT_FUNDS: u32   = 6002;
pub const ERR_GAME_ALREADY_OVER: u32    = 6003;
pub const ERR_NOT_YOUR_TURN: u32        = 6004;
pub const ERR_INVALID_MOVE: u32         = 6005;
```

> **🔄 Coming from Solidity?** In Solidity, you define custom errors with `error MyError(uint256 param)` and revert with `revert MyError(value)`. In LEZ, custom errors are `u32` codes wrapped in `LezError::Custom(...)`. There is currently no support for attaching structured data to a custom error — if you need to communicate error details, log them via `spel_framework::log!` before returning the error.

> **💡 Tip:** Export your error constants from a dedicated module and keep them in sync with your IDL's error section. Clients can use these constants to display human-readable error messages.

---

## Error Handling Patterns

### Chaining Results

Use `map_err` to convert standard errors to `LezError`:

```rust
let data = MyState::try_from_slice(&account.data)
    .map_err(|_| LezError::DeserializationError)?;

let serialized = updated_state.try_to_vec()
    .map_err(|_| LezError::SerializationError)?;
```

### Early Returns

Use the `?` operator for clean error propagation:

```rust
fn validate_player(player: &AccountWithMetadata, expected_owner: &[u8; 32]) -> Result<PlayerState, LezError> {
    if player.owner != *expected_owner {
        return Err(LezError::InvalidAccountOwner);
    }
    PlayerState::try_from_slice(&player.data)
        .map_err(|_| LezError::DeserializationError)
}

#[instruction]
pub fn move_piece(
    board: AccountWithMetadata,
    player: AccountWithMetadata,
    from_x: u32,
    from_y: u32,
    to_x: u32,
    to_y: u32,
) -> LezResult {
    let player_state = validate_player(&player, &EXPECTED_PROGRAM_ID)?;
    // ...
}
```

---

## Error Code Reference Summary

| Code Range | Source | Description |
|-----------|--------|-------------|
| 1000–1999 | LEZ Framework | Account constraints, ownership, signing |
| 2000–5999 | Reserved | Future framework use |
| 6000+ | User-defined | Program-specific domain errors |
| `SerializationError` | LEZ Framework | Borsh write failure (no numeric code) |
| `DeserializationError` | LEZ Framework | Borsh read failure (no numeric code) |

> **⚠️ Warning:** Do not define custom error codes below 6000. The range 2000–5999 is reserved for future LEZ framework use. A custom error code that collides with a future framework code will be impossible to distinguish at the client level.

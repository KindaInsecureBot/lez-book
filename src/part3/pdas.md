# PDAs (Program Derived Addresses)

## What Are PDAs?

A PDA is an account whose address is derived deterministically from seeds. Given the same seeds, you always get the same address. This is how you implement patterns like "the balance account FOR user X" or "the metadata FOR token Y."

> **🔄 Coming from Solidity?** PDAs are conceptually similar to `CREATE2` deterministic addresses or mapping keys. `mapping(address => Balance) balances` in Solidity becomes a PDA where the seeds include the user's address. The key difference: PDAs are full accounts with state, not just values in a mapping.

## Seed Types

SPEL supports three seed types:

```rust
// Literal seed — a constant string
#[account(pda = literal("counter"))]

// Account seed — derive from another account's ID
#[account(pda = account("user"))]

// Arg seed — derive from an instruction argument
#[account(pda = arg("key"))]
```

## Multi-Seed PDAs

Combine seeds with an array:

```rust
// "vault" + user account ID → vault PDA for this user
#[account(pda = [literal("vault"), account("user")])]

// "tile" + x coordinate + y coordinate → hex tile PDA
#[account(pda = [literal("tile"), arg("x"), arg("y")])]
```

The derivation uses SHA-256 over all seeds concatenated (with the program ID mixed in).

## PDA Usage Example: Per-User State

Mapping equivalent: `mapping(address => PlayerState) players`

```rust
#[derive(BorshSerialize, BorshDeserialize, Default, Clone)]
pub struct PlayerState {
    pub tile_count: u64,
    pub joined_at: u64,
}

#[instruction]
pub fn join(
    #[account(init, pda = account("player"))] player_state: AccountWithMetadata<PlayerState>,
    #[account(signer)] player: AccountWithMetadata<()>,
) -> Result<LezOutput, LezError> {
    let mut state = PlayerState::default();
    state.joined_at = 0; // set timestamp if available
    state.tile_count = 0;

    let updated = player_state.with_data(state);
    Ok(LezOutput::states_only(vec![
        AccountPostState::new_claimed(updated),
        AccountPostState::new(player),
    ]))
}
```

The `player_state` account's address is derived from the `player` account's ID. You don't compute this manually — the SPEL macro and protocol handle it.

## Multi-Seed Example: logos-land Hex Tiles

From the logos-land program — each hex tile is a PDA keyed by `(x, y)` coordinates:

```rust
// Hex tile state
#[derive(BorshSerialize, BorshDeserialize, Default, Clone)]
pub struct TileState {
    pub owner_commitment: [u8; 32], // SHA256 of owner pubkey (for privacy)
    pub claimed_at: u64,
    pub terrain: u8,
    pub elevation: u8,
}

#[instruction]
pub fn claim(
    #[account(init, pda = [literal("tile"), arg("x"), arg("y")])] tile: AccountWithMetadata<TileState>,
    #[account(signer)] player: AccountWithMetadata<()>,
    x: u64, // biased coords (signed coords stored as u64 with bias)
    y: u64,
) -> Result<LezOutput, LezError> {
    // ...
}
```

## Signed Coordinates as u64 Seeds

When using signed coordinates as PDA seeds, you need to bias them:

```rust
// In logos-land: signed coordinates (i64) stored as u64 with bias
const COORD_BIAS: u64 = 1 << 31;

fn to_seed(coord: i64) -> u64 {
    (coord as i64 + COORD_BIAS as i64) as u64
}
```

This ensures negative coordinates don't cause issues with seed derivation.

## The lez-cli pda Bug

> **⚠️ Warning:** `lez-cli pda` gives WRONG addresses for `u64` and `u128` arg seeds. It has a serialization discrepancy with the on-chain derivation. When building tooling that needs to pre-compute PDA addresses with numeric seeds, compute them yourself using SHA-256 matching the on-chain logic, or derive them from the deployed program's behavior. Literal and account seeds work correctly.

## Computing PDAs Manually

If you need to compute a PDA address off-chain (e.g., for a client):

```rust
use sha2::{Sha256, Digest};

fn derive_pda(program_id: &[u8; 32], seeds: &[&[u8]]) -> [u8; 32] {
    let mut hasher = Sha256::new();
    hasher.update(program_id);
    for seed in seeds {
        hasher.update(seed);
    }
    hasher.finalize().into()
}

// Example: derive tile PDA for (x=5, y=3)
let x: u64 = to_seed(5);
let y: u64 = to_seed(3);
let pda = derive_pda(&PROGRAM_ID, &[b"tile", &x.to_le_bytes(), &y.to_le_bytes()]);
```

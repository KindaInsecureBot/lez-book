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

## Precise Derivation Rules

Understanding the exact derivation is important when computing PDA addresses off-chain.

**Each seed is zero-padded to exactly 32 bytes:**
- String literals: UTF-8 encoded, zero-padded to 32 bytes. Strings longer than 32 bytes are truncated to 32 bytes.
- Account IDs: already 32 bytes, no padding needed.
- Numeric args: serialized as little-endian, zero-padded to 32 bytes.

**Multi-seed derivation concatenates the padded seeds, then hashes:**
```
seed1_padded = seed1 || 0x00 * (32 - len(seed1))  // pad to 32 bytes
seed2_padded = seed2 || 0x00 * (32 - len(seed2))

combined_seed = SHA-256(seed1_padded || seed2_padded || ...)
```
The combined seed is a single 32-byte value.

**Program ID format**: The program ID used in PDA derivation is a `[u32; 8]` — the RISC Zero ImageID as eight 32-bit words, not a flat `[u8; 32]`. This is important when computing PDA addresses manually in client code.

**Final derivation**:
```
PDA = SHA-256(program_id_as_[u32;8] || combined_seed)
```

## PDA Usage Example: Per-User State

Mapping equivalent: `mapping(address => PlayerState) players`

```rust
// PlayerState with manual serialization (borsh_derive doesn't compile for riscv32im)
pub struct PlayerState {
    pub tile_count: u64,
    pub joined_at: u64,
}

impl PlayerState {
    pub const SIZE: usize = 8 + 8;
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut buf = Vec::with_capacity(Self::SIZE);
        buf.extend_from_slice(&self.tile_count.to_le_bytes());
        buf.extend_from_slice(&self.joined_at.to_le_bytes());
        buf
    }
}

#[instruction]
pub fn join(
    #[account(init, pda = account("player"))] player_state: AccountWithMetadata,
    #[account(signer)] player: AccountWithMetadata,
) -> LezResult {
    let state = PlayerState {
        tile_count: 0,
        joined_at: 0,
    };

    let mut new_account = player_state.account.clone();
    new_account.data = state.to_bytes().try_into().unwrap();

    Ok(LezOutput::states_only(vec![
        AccountPostState::new_claimed(new_account),
        AccountPostState::new(player.account.clone()),
    ]))
}
```

The `player_state` account's address is derived from the `player` account's ID. You don't compute this manually — the SPEL macro and protocol handle it.

## Multi-Seed Example: logos-land Hex Tiles

From the logos-land program — each hex tile is a PDA keyed by `(x, y)` coordinates:

```rust
// Hex tile state — manual serialization, no derives
pub struct TileState {
    pub owner_commitment: [u8; 32], // SHA256 of owner pubkey (for privacy)
    pub claimed_at: u64,
    pub terrain: u8,
    pub elevation: u8,
}

impl TileState {
    pub const SIZE: usize = 32 + 8 + 1 + 1;
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut buf = Vec::with_capacity(Self::SIZE);
        buf.extend_from_slice(&self.owner_commitment);
        buf.extend_from_slice(&self.claimed_at.to_le_bytes());
        buf.push(self.terrain);
        buf.push(self.elevation);
        buf
    }
}

#[instruction]
pub fn claim(
    #[account(init, pda = [literal("tile"), arg("x"), arg("y")])] tile: AccountWithMetadata,
    #[account(signer)] player: AccountWithMetadata,
    x: u64, // biased coords (signed coords stored as u64 with bias)
    y: u64,
) -> LezResult {
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

## The spel pda Bug

> **⚠️ Warning:** `spel pda` gives WRONG addresses for `u64` and `u128` arg seeds. It has a serialization discrepancy with the on-chain derivation. When building tooling that needs to pre-compute PDA addresses with numeric seeds, compute them yourself using SHA-256 matching the on-chain logic, or derive them from the deployed program's behavior. Literal and account seeds work correctly.

## Computing PDAs Manually

If you need to compute a PDA address off-chain (e.g., for a client), you must match the exact on-chain derivation: pad each seed to 32 bytes, hash the concatenation, then hash with the program ID.

```rust
use sha2::{Sha256, Digest};

/// Pad a seed to exactly 32 bytes (truncate if longer)
fn pad_seed(seed: &[u8]) -> [u8; 32] {
    let mut padded = [0u8; 32];
    let len = seed.len().min(32);
    padded[..len].copy_from_slice(&seed[..len]);
    padded
}

/// Derive a PDA from a program ID (as [u32; 8]) and a list of seeds
fn derive_pda(program_id: &[u32; 8], seeds: &[&[u8]]) -> [u8; 32] {
    // Step 1: if multiple seeds, hash their concatenation into one 32-byte seed
    let combined_seed: [u8; 32] = if seeds.len() == 1 {
        pad_seed(seeds[0])
    } else {
        let mut hasher = Sha256::new();
        for seed in seeds {
            hasher.update(pad_seed(seed));
        }
        hasher.finalize().into()
    };

    // Step 2: hash program_id (as bytes) + combined_seed
    let mut hasher = Sha256::new();
    for word in program_id {
        hasher.update(word.to_le_bytes());
    }
    hasher.update(combined_seed);
    hasher.finalize().into()
}

// Example: derive tile PDA for (x=5, y=3)
let program_id: [u32; 8] = COUNTER_IMAGE_ID; // from your compiled program

let x: u64 = to_seed(5);
let y: u64 = to_seed(3);

let pda = derive_pda(
    &program_id,
    &[b"tile", &x.to_le_bytes(), &y.to_le_bytes()],
);
```

> **💡 Tip:** When debugging PDA mismatches, log both the expected address (computed off-chain) and the actual address (from the sequencer or a known transaction). A mismatch usually means the seed encoding or padding differs.

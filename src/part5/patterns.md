# Patterns & Recipes

This chapter collects common patterns you'll reach for repeatedly when writing LEZ programs. Each is battle-tested from real programs (primarily logos-land).

---

## Init-If-Needed Pattern

Sometimes you want to initialize an account the first time it's used, or update it if it already exists. Use `new_claimed_if_default`:

```rust
#[instruction]
pub fn ensure_player_state(
    #[account(mut)] player_state: AccountWithMetadata,
    #[account(signer)] player: AccountWithMetadata,
) -> LezResult {
    let new_data = if player_state.account.data.is_empty() {
        // First time: initialize
        let state = PlayerState { tile_count: 0, joined_at: 0 };
        state.to_bytes()
    } else {
        // Already exists: update
        let mut state = PlayerState::from_bytes(&player_state.account.data).unwrap();
        state.tile_count += 1;
        state.to_bytes()
    };

    let mut updated = player_state.account.clone();
    updated.data = new_data.try_into().unwrap();

    Ok(LezOutput::states_only(vec![
        AccountPostState::new_claimed_if_default(updated),
        AccountPostState::new(player.account.clone()),
    ]))
}
```

Use `new_claimed_if_default` when the account may or may not exist — the sequencer will create it if it's being initialized for the first time.

> **💡 Tip:** `new_claimed_if_default` is the idiomatic "upsert" in LEZ. It eliminates the need for a separate `initialize` instruction in simple cases.

---

## Owner-Only Access Pattern

```rust
#[instruction]
pub fn admin_action(
    #[account(mut, owner = MY_PROGRAM_ID)] config: AccountWithMetadata,
    #[account(signer)] caller: AccountWithMetadata,
) -> LezResult {
    // Deserialize config to check the stored admin field
    let config_state = ConfigState::from_bytes(&config.account.data).unwrap();

    // Manual ownership check — verify the caller is the stored admin
    if config_state.admin != caller.account.id {
        return Err(LezError::Unauthorized);
    }
    // ... admin logic ...
    Ok(LezOutput::states_only(vec![
        AccountPostState::new(config.account.clone()),
        AccountPostState::new(caller.account.clone()),
    ]))
}
```

`#[account(owner = MY_PROGRAM_ID)]` checks that the account was created by your program. The additional manual check on `config_state.admin` (deserialized from raw bytes) verifies that the specific caller is the designated admin.

> **🔄 Coming from Solidity?** In Solidity, `onlyOwner` is a modifier on the contract. In LEZ, "ownership" has two layers: the account's `program_owner` (which program created it) and your application-level admin stored in the account data. Check both as needed.

---

## PDA as State Key Pattern

The canonical pattern for "state per user" — like a mapping:

```rust
#[instruction]
pub fn create_user_profile(
    #[account(init, pda = account("user"))] profile: AccountWithMetadata,
    #[account(signer)] user: AccountWithMetadata,
    username_hash: [u8; 32],
) -> LezResult {
    let state = UserProfile {
        username_hash,
        created_at: 0,
    };
    let mut new_profile = profile.account.clone();
    new_profile.data = state.to_bytes().try_into().unwrap();
    Ok(LezOutput::states_only(vec![
        AccountPostState::new_claimed(new_profile),
        AccountPostState::new(user.account.clone()),
    ]))
}
```

The profile's address is `SHA256(PROGRAM_ID || user.id())`. Every user gets exactly one profile, and the address is deterministic.

> **💡 Tip:** Because PDA addresses are deterministic, clients can compute the expected profile address without querying the chain. This is a major advantage over contract-internal mappings.

---

## Multi-PDA Pattern (logos-land style)

Tracking both a per-user summary AND per-tile state in a single instruction:

```rust
#[instruction]
pub fn claim_tile(
    #[account(init, pda = [literal("tile"), arg("x"), arg("y")])]
        tile: AccountWithMetadata,
    #[account(mut, pda = account("player"))]
        player_state: AccountWithMetadata,
    #[account(signer)] player: AccountWithMetadata,
    x: u64,
    y: u64,
) -> LezResult {
    // Build tile data
    let tile_data = TileState {
        owner_commitment: sha256(&player.account.id),
        claimed_at: 0,
        terrain: deterministic_terrain(x, y),
        elevation: deterministic_elevation(x, y),
    };
    let mut new_tile = tile.account.clone();
    new_tile.data = tile_data.to_bytes().try_into().unwrap();

    // Update player summary
    let mut ps = PlayerState::from_bytes(&player_state.account.data).unwrap();
    ps.tile_count += 1;
    let mut updated_player_state = player_state.account.clone();
    updated_player_state.data = ps.to_bytes().try_into().unwrap();

    Ok(LezOutput::states_only(vec![
        AccountPostState::new_claimed(new_tile),
        AccountPostState::new(updated_player_state),
        AccountPostState::new(player.account.clone()),
    ]))
}
```

This pattern lets one instruction atomically update multiple PDA-derived accounts. The tile address is `SHA256(PROGRAM_ID || "tile" || x_le_bytes || y_le_bytes)` and the player-state address is `SHA256(PROGRAM_ID || player.id())`.

---

## External Instruction Enum for Shared Types

When your program uses types shared with clients or other programs, define the instruction enum externally and reference it:

```rust
// In a shared crate (shared-types/src/lib.rs)
#[derive(BorshSerialize, BorshDeserialize)]
pub enum CounterInstruction {
    Initialize,
    Increment,
    GetCount,
}
```

Reference in your SPEL program:

```rust
use shared_types::CounterInstruction;

// The SPEL macro can use this for dispatch
```

This pattern is useful when you have a CLI wrapper crate or client SDK that also needs to serialize instructions — sharing the type definition prevents encoding divergence.

---

## Manual Borsh Serialization

`borsh_derive` doesn't compile for `riscv32im`. Use `borsh`'s built-in derive feature:

```toml
# In your guest/Cargo.toml
[dependencies]
borsh = { version = "1", features = ["derive"] }
```

```rust
use borsh::{BorshSerialize, BorshDeserialize};

// This works on riscv32im (using borsh's derive, not borsh_derive)
#[derive(BorshSerialize, BorshDeserialize, Default, Clone)]
pub struct MyState {
    pub value: u64,
    pub data: [u8; 32],
}
```

If you hit derive issues, implement manually:

```rust
impl BorshSerialize for MyState {
    fn serialize<W: borsh::io::Write>(&self, writer: &mut W) -> borsh::io::Result<()> {
        self.value.serialize(writer)?;
        self.data.serialize(writer)?;
        Ok(())
    }
}
```

> **⚠️ Warning:** Never add `borsh_derive` as a separate dependency in guest code. Only use `borsh = { version = "1", features = ["derive"] }`. See the Gotchas chapter for details.

---

## Deterministic Properties from Coordinates (logos-land)

When tile or entity properties should be fixed by their coordinates, derive them deterministically from a hash rather than storing them:

```rust
// Deterministic terrain/elevation from coordinates
fn tile_properties(x: u64, y: u64) -> (u8, u8) {
    use sha2::{Sha256, Digest};
    let mut h = Sha256::new();
    h.update(&x.to_le_bytes());
    h.update(&y.to_le_bytes());
    let hash = h.finalize();
    let terrain = hash[0] % 5;    // 5 terrain types
    let elevation = hash[1] % 10; // 0-9 elevation
    (terrain, elevation)
}
```

This gives each tile deterministic properties based on coordinates — no need to store them explicitly. Any client can compute the same value without reading chain state.

> **💡 Tip:** This pattern is ideal for "world generation" in on-chain games. Storing terrain for a large map would be prohibitively expensive; deriving it on-demand is free.

---

## Privacy-Safe Identity Storage

When storing a user's identity in a public account, never store the raw public key if you want to hide the relationship. Use a commitment:

```rust
// ❌ Stores raw identity — linkable on-chain
pub struct TileState {
    pub owner: [u8; 32], // raw pubkey, visible to all
}

// ✅ Stores a commitment — hides the owner
pub struct TileState {
    pub owner_commitment: [u8; 32], // SHA256(owner_pubkey || salt)
}
```

Only the owner (who knows their own pubkey and salt) can prove they own the tile. Third parties see only the commitment.

---

## Rest Accounts Pattern

For instructions that operate on a variable number of accounts:

```rust
#[instruction]
pub fn batch_update(
    #[account(signer)] authority: AccountWithMetadata,
    #[rest] targets: Vec<AccountWithMetadata>,
) -> LezResult {
    let mut post_states = vec![AccountPostState::new(authority.account.clone())];

    for target in targets {
        // Deserialize, update, and write back
        let mut data = TargetState::from_bytes(&target.account.data).unwrap();
        data.last_updated = 0;
        let mut updated = target.account.clone();
        updated.data = data.to_bytes().try_into().unwrap();
        post_states.push(AccountPostState::new(updated));
    }

    Ok(LezOutput::states_only(post_states))
}
```

Pass multiple accounts from the CLI with repeated `--targets` flags:

```bash
lez-cli call --idl program.json \
  --instruction batch_update \
  --authority <addr> \
  --targets <addr1> \
  --targets <addr2> \
  --targets <addr3>
```

> **⚠️ Warning:** When calling instructions with `Vec<AccountWithMetadata>` (rest accounts), check your IDL for the exact CLI flag name. See the Gotchas chapter.

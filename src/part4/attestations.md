# Private State Attestations

## What Are Attestations?

An attestation is a ZK proof that some fact about your private state is true, without revealing the state itself.

Example: "My private balance is ≥ 1000 tokens" — proven without revealing the actual balance.

> **🔄 Coming from Solidity?** There's no EVM equivalent. This is ZK cryptography applied to smart contract state. The closest analogue is a SNARK generated off-chain and verified by a contract — but here, it's built into the protocol and works with any program's state.

## The Balance Attestation Example

From our real `balance-attestation` program:

**What it proves**: caller's private token balance ≥ threshold
**What it reveals**: nothing (not the actual balance, not the account address)

> **⚠️ Note on serialization:** The code below shows the logical structure. In practice, `TokenBalance` uses manual serialization (not `borsh_derive`) because `borsh_derive` proc macros don't compile for the `riscv32im` guest target. See the Accounts & State chapter for the manual serialization pattern.

```rust
#[lez_program]
mod balance_attestation {
    use super::*;

    #[instruction]
    pub fn attest_balance(
        balance_account: AccountWithMetadata<TokenBalance>,
        threshold: u64,
    ) -> Result<LezOutput, LezError> {
        let balance = balance_account.data().amount;

        if balance < threshold {
            return Err(LezError::Custom(1)); // attestation fails
        }

        // Success: caller's balance >= threshold
        Ok(LezOutput::states_only(vec![
            AccountPostState::new(balance_account),
        ]))
    }
}
```

When run as a privacy-preserving transaction:
- The `balance_account` is decrypted by the privacy circuit locally in the wallet
- Your program checks `balance >= threshold`
- If true, a ZK proof is generated: "this program ran successfully on this private state"
- The proof goes on-chain; the actual balance never appears anywhere

## Real Example: logos-land Attestations

From the logos-land hex grid program — three types of attestations:

**1. attest_ownership** — prove you own a specific tile:

```rust
#[instruction]
pub fn attest_ownership(
    tile: AccountWithMetadata<TileState>,
    #[account(signer)] player: AccountWithMetadata<()>,
    x: u64,
    y: u64,
) -> Result<LezOutput, LezError> {
    let state = tile.data();
    // Owner stored as SHA256(pubkey) for privacy — compare commitment
    let owner_commitment = sha256(player.id());

    if state.owner_commitment != owner_commitment {
        return Err(LezError::Custom(2)); // not the owner
    }

    // Proof: "player owns this tile" — without revealing which pubkey owns which tile
    Ok(LezOutput::states_only(vec![
        AccountPostState::new(tile),
        AccountPostState::new(player),
    ]))
}
```

**2. attest_connected** — BFS to prove N connected tiles in zkVM:

```rust
#[instruction]
pub fn attest_connected(
    tiles: Vec<AccountWithMetadata<TileState>>,
    #[account(signer)] player: AccountWithMetadata<()>,
    min_connected: u64,
) -> Result<LezOutput, LezError> {
    // BFS runs INSIDE the zkVM — the proof covers the entire graph traversal
    let player_commitment = sha256(player.id());
    let owned_tiles: Vec<_> = tiles.iter()
        .filter(|t| t.data().owner_commitment == player_commitment)
        .collect();

    let max_component = bfs_max_component(&owned_tiles);

    if max_component < min_connected as usize {
        return Err(LezError::Custom(3)); // not enough connected tiles
    }

    let post_states: Vec<_> = tiles.into_iter()
        .map(AccountPostState::new)
        .chain(std::iter::once(AccountPostState::new(player)))
        .collect();
    Ok(LezOutput::states_only(post_states))
}
```

The BFS runs entirely inside the RISC Zero zkVM. The ZK proof proves not just the final result but the entire computation — including every step of the graph traversal.

**3. attest_islands** — count connected components:

```rust
#[instruction]
pub fn attest_islands(
    tiles: Vec<AccountWithMetadata<TileState>>,
    #[account(signer)] player: AccountWithMetadata<()>,
) -> Result<LezOutput, LezError> {
    let player_commitment = sha256(player.id());
    let owned: Vec<_> = tiles.iter()
        .filter(|t| t.data().owner_commitment == player_commitment)
        .collect();

    let island_count = count_connected_components(&owned);

    // island_count is committed to the ZK proof output
    // Verifier sees: "this player has N islands" without seeing which tiles

    let post_states: Vec<_> = tiles.into_iter()
        .map(AccountPostState::new)
        .chain(std::iter::once(AccountPostState::new(player)))
        .collect();
    Ok(LezOutput::states_only(post_states))
}
```

## How the Privacy Circuit Wraps Your Program

The execution flow for an attestation:

```
1. User submits: program_id + instruction + encrypted private accounts
2. Wallet decrypts accounts locally using nsk
3. Privacy circuit + your program run together in the zkVM
4. Your instruction executes with decrypted data
5. If instruction succeeds → attestation valid
6. ZK proof generated: covers decryption, your logic, and privacy constraints
7. Proof submitted to sequencer
8. Sequencer verifies proof, records attestation on-chain
```

Your code is step 4. Everything else is the protocol.

## Working Setup: Running Private Attestations

`lez-cli` cannot sign for private accounts — you must use the wallet Rust API. Here is a complete working workflow:

```bash
# 1. Build lssa from the main branch
#    NOTE: do NOT use commit 767b5af — it has a broken commitment RPC
cd ~/lssa
git checkout main
cargo build --release \
  -p sequencer_runner \
  -p wallet \
  --features standalone \
  --jobs 2

# 2. Clean any stale state and start the sequencer
rm -rf rocksdb/
./target/release/sequencer_runner sequencer_runner/configs/debug &

# 3. Create a private account and initialize its commitment
wallet account new private
# Output: Private/ABC123...

wallet auth-transfer init --account-id Private/ABC123...

# 4. Wait for the init tx to land, then sync
sleep 15
wallet account sync-private

# 5. Fund the private account from a genesis account
wallet auth-transfer send \
  --from Private/E8Hwi... \
  --to Private/ABC123... \
  --amount 1000

# 6. Sync again to pick up the incoming transfer
wallet account sync-private

# 7. Deploy your program
lez-cli deploy --program ./target/riscv-guest/release/balance_attestation

# 8. Run the attestation via wallet Rust API (see below)
```

In your client code:

```rust
use nssa::program::Program;
use nssa::privacy_preserving_transaction::circuit::ProgramWithDependencies;

// Load program binary
let bytecode = std::fs::read("target/riscv-guest/release/balance_attestation")?;
let program = Program::new(bytecode)?;
let program_with_deps = ProgramWithDependencies::from(program);

// instruction_index 0 = attest_balance, threshold = 500
let instruction_data = risc0_zkvm::serde::to_vec(&(0u32, 500u64))?;

let result = wallet.send_privacy_preserving_tx(
    vec![
        PrivacyPreservingAccount::PrivateOwned(balance_account_id),
        PrivacyPreservingAccount::PrivateOwned(player_account_id),
    ],
    instruction_data,
    &program_with_deps,
).await?;

println!("Attestation tx: {:?}", result);
```

## When to Use Attestations vs Public Transactions

Use **attestations** when:
- The fact being proven matters, but the underlying data should stay private
- Example: prove creditworthiness (balance ≥ threshold) for a loan
- Example: prove land ownership for voting rights without revealing which plots

Use **public transactions** when:
- The state change itself is the goal, and visibility is fine
- Example: claiming a tile in a game where ownership is meant to be public

> **💡 Tip:** You can mix and match. A game might use public transactions for land claims (ownership is public) but private attestations for proving accumulated holdings (exact portfolio is private).

> **⚠️ Warning:** Private transactions are CPU-intensive — the ZK proof is generated locally on your machine. For attestations with large `Vec<AccountWithMetadata<T>>` inputs (like BFS over many tiles), expect significant proving time. Design your attestation instructions to take the minimum necessary accounts.

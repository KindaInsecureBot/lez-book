# Private State Attestations

## What Are Attestations?

An attestation is a ZK proof that some fact about your private state is true, without revealing the state itself.

Example: "My private balance is ≥ 1000 tokens" — proven without revealing the actual balance.

> **🔄 Coming from Solidity?** There's no EVM equivalent. This is ZK cryptography applied to smart contract state. The closest analogue is a SNARK generated off-chain and verified by a contract — but here, it's built into the protocol and works with any program's state.

## The Balance Attestation Example

From our real `balance-attestation` program:

**What it proves**: caller's private token balance ≥ threshold
**What it reveals**: nothing (not the actual balance, not the account address)

The program:

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
- The `balance_account` is decrypted by the privacy circuit
- Your program checks `balance >= threshold`
- If true, a ZK proof is generated: "this program ran successfully on this private state"
- The proof goes on-chain; the balance never appears anywhere

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
    // Owner stored as SHA256(pubkey) for privacy
    let owner_commitment = sha256(player.id());

    if state.owner_commitment != owner_commitment {
        return Err(LezError::Custom(2)); // not the owner
    }

    // Prove ownership without revealing pubkey-tile link
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
    // BFS runs INSIDE the zkVM — proof generation includes the graph traversal
    let owned_tiles: Vec<_> = tiles.iter()
        .filter(|t| t.data().owner_commitment == sha256(player.id()))
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

Note: The BFS runs entirely inside the RISC Zero zkVM. The ZK proof proves not just the final result but the entire computation.

**3. attest_islands** — count connected components:

```rust
#[instruction]
pub fn attest_islands(
    tiles: Vec<AccountWithMetadata<TileState>>,
    #[account(signer)] player: AccountWithMetadata<()>,
) -> Result<LezOutput, LezError> {
    let owned: Vec<_> = tiles.iter()
        .filter(|t| t.data().owner_commitment == sha256(player.id()))
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
2. Privacy circuit decrypts accounts
3. Your program instruction runs with decrypted data
4. If instruction succeeds, output = "attestation valid"
5. ZK proof generated covering steps 2-4
6. Proof submitted to sequencer
7. Sequencer verifies proof, records attestation on-chain
```

Your code is step 3. Everything else is the protocol.

## The Rust API for Private Transactions

```rust
// From wallet/client code
use lssa_client::WalletClient;

let client = WalletClient::new("http://localhost:3040");

client.send_privacy_preserving_tx(
    &program_id,
    "attest_balance",
    accounts,          // encrypted
    &args_bytes,       // borsh-encoded args
    visibility_mask,   // which outputs to encrypt
).await?;
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

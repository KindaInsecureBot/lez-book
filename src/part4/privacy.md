# Privacy-Preserving Transactions

## The Killer Feature

LEZ supports two types of transactions:
1. **Public transactions** — like normal blockchain txs, all state visible
2. **Privacy-preserving transactions** — accounts encrypted, balances hidden, ownership private

The remarkable thing: **the same program binary handles both**. You don't write separate "private" and "public" versions of your program. The privacy circuit wraps your program at the protocol level.

> **🔄 Coming from Solidity?** Nothing like this exists in stock EVM. Tornado Cash adds privacy for token mixing. Aztec builds a separate privacy L2. LEZ bakes privacy into every program at the account model level. Your Counter program automatically supports private transactions — you didn't write any ZK code.

## Transaction Visibility Matrix

| From | To | Sender | Receiver | Amount | Program |
|---|---|---|---|---|---|
| Public account | Public account | Visible | Visible | Visible | Visible |
| Public account | Private account | Visible | Hidden | Hidden | Visible |
| Private account | Public account | Hidden | Visible | Hidden | Visible |
| Private account | Private account | Hidden | Hidden | Hidden | Visible |

The program being called is always visible (it's the `program_id`). What's hidden is who is calling it and what accounts are involved.

## Private Account Key Chain

Private accounts use a derived key hierarchy. Here are the precise formulas:

```
nsk  (NullifierSecretKey) — 32 bytes, never leaves your machine
  │
  ├── npk = SHA256("LEE/keys" || nsk || 0x07 || 0x00*23)
  │         (NullifierPublicKey — share this so others can send you funds)
  │
  ├── vpk = secp256k1_pubkey(nsk)
  │         (ViewingPublicKey — share to let someone audit your history)
  │
  └── ssk = ephemeral ECDH key, derived per-transaction
            (SharedSecretKey — used to encrypt post-states)
```

**Private account IDs** are derived from the nullifier public key:
```
account_id = SHA256("/LEE/v0.3/AccountId/Private/" || npk)
```

- **nsk**: your private key for nullifier derivation. Never share this.
- **npk**: your public key for commitments. Others use this to send you private funds.
- **vpk**: for view-key sharing — lets someone scan the chain to see your private txs without spending them.
- **ssk**: ephemeral signing key, freshly derived each transaction.

## Commitment Scheme

Private account state is hidden using a commitment scheme. When you receive private funds or state, a **commitment** goes on-chain:

```
Commitment = SHA256(npk || program_owner || balance || nonce || SHA256(data))
```

The commitment hides all account details. Commitments live in a Merkle tree. To spend a commitment, you prove membership in the tree with a Merkle proof — without revealing which leaf you're proving.

The protocol stores commitments as a flat append-only list. The `CommitmentSetDigest` is the root of this Merkle tree at the time of spending.

## Nullifier Scheme

When you spend a commitment (transfer out, invoke logic), a **nullifier** goes on-chain:

```
nullifier = f(nsk, old_commitment)
```

The nullifier proves the old state was consumed — without revealing which commitment it corresponds to. The link between commitment and nullifier is only computable with `nsk`.

The sequencer tracks all nullifiers and **rejects duplicate nullifiers** — this prevents double-spending. Since the nullifier is deterministic from `nsk + commitment`, you can't spend the same state twice.

## View Tags

View tags let your wallet efficiently scan the chain for incoming transactions without doing full decryption on every tx:

```
ViewTag = SHA256("/NSSA/v0.2/ViewTag/" || npk || vpk)[0]  // first byte only
```

Your wallet computes the view tag for each transaction and compares it to the stored byte. If it doesn't match — which is 99.6% of all transactions — skip. If it matches, do the full ECDH decryption to check if it's yours.

This gives efficient wallet scanning without revealing which transactions are yours to observers.

## Visibility Mask

Each account in a privacy-preserving transaction has a visibility level encoded as a small integer:

| Value | Name | Meaning |
|---|---|---|
| `0` | Public | Plaintext on chain, anyone can read |
| `1` | PrivateAuth | Encrypted, only signer (owner) can decrypt |
| `2` | PrivateUnauth | Encrypted for recipient, using their `npk`+`vpk` |

The calling program does NOT set visibility explicitly — the wallet API sets it based on which accounts are private/public and whether they're owned or foreign.

## Privacy-Preserving Transaction Structure

The wire format of a privacy-preserving transaction:

```rust
PrivacyPreservingTransaction {
    message: Message {
        // Public account IDs involved (public accounts stay visible)
        public_account_ids: Vec<AccountId>,
        // Nonces for replay protection
        nonces: Vec<Nonce>,
        // Public accounts' new states (plaintext)
        public_post_states: Vec<Account>,
        // Private accounts' new states (encrypted)
        encrypted_private_post_states: Vec<EncryptedAccountData>,
        // New commitments added to the commitment tree
        new_commitments: Vec<Commitment>,
        // Nullifiers: (nullifier, tree_root_at_spend_time)
        new_nullifiers: Vec<(Nullifier, CommitmentSetDigest)>,
    },
    witness_set: WitnessSet {
        // Signatures authorizing the transaction
        signatures_and_public_keys: Vec<(Signature, PublicKey)>,
        // RISC Zero receipt proving correct execution + privacy circuit
        proof: Proof,
    }
}
```

## Wallet Operations for Private Accounts

These are the real, tested wallet commands:

```bash
# Create a new private account (generates nsk, npk, vpk locally)
wallet account new private
# Output: Private/<base58_account_id>

# Initialize: register the commitment in the Merkle tree
# REQUIRED before the first transaction — even genesis/preconfigured accounts
wallet auth-transfer init --account-id Private/<base58>

# Sync private state: scan blockchain for commitments belonging to you
# ALWAYS run this before private transfers — wallet needs fresh membership proofs
wallet account sync-private

# Transfer: public → private (fund your private account)
wallet auth-transfer send \
  --from Public/<base58> \
  --to Private/<base58> \
  --amount 1000

# Transfer: private → private (fully private, no on-chain link)
wallet auth-transfer send \
  --from Private/<sender_base58> \
  --to Private/<receiver_base58> \
  --amount 300

# Transfer to a foreign private account (by npk + vpk, when you don't have their account ID)
wallet auth-transfer send \
  --from Private/<base58> \
  --to-npk <hex_npk> --to-vpk <hex_vpk> \
  --amount 100
```

> **⚠️ Warning:** Always run `wallet account sync-private` before any private transfer. The wallet needs current Merkle membership proofs for your commitments. If the proofs are stale, the transaction will fail at the sequencer.

> **⚠️ Warning:** Always run `wallet auth-transfer init` before first use. This registers your initial commitment in the Merkle tree. Even if a genesis configuration pre-funded your private account with a balance, the commitment won't exist until you run init.

## Using Custom Programs with Private Transactions

`lez-cli` **cannot sign for private accounts**. For custom programs that touch private accounts, you must use the wallet Rust API directly.

```rust
use nssa::program::Program;
use nssa::privacy_preserving_transaction::circuit::ProgramWithDependencies;

// Load your compiled SPEL program binary
let bytecode = std::fs::read("target/riscv-guest/release/my_program")?;
let program = Program::new(bytecode)?;
let program_with_deps = ProgramWithDependencies::from(program);

// Serialize instruction data: instruction_index as u32 first, then args
// instruction_index is the 0-based index of the instruction in your program
let instruction_index: u32 = 0;
let threshold: u128 = 1000;
let instruction_data = risc0_zkvm::serde::to_vec(&(instruction_index, threshold))?;

// Submit the privacy-preserving transaction
let result = wallet.send_privacy_preserving_tx(
    vec![PrivacyPreservingAccount::PrivateOwned(account_id)],
    instruction_data,
    &program_with_deps,
).await?;
```

### PrivacyPreservingAccount Variants

| Variant | When to use |
|---|---|
| `PrivacyPreservingAccount::Public(id)` | Public account (visible on-chain, passes through as-is) |
| `PrivacyPreservingAccount::PrivateOwned(id)` | Private account you own — wallet decrypts it for the zkVM |
| `PrivacyPreservingAccount::PrivateForeign { npk, vpk }` | Private account you don't own — only receives encrypted output |

## How the Privacy Circuit Wraps Your Program

The privacy circuit is a second RISC Zero program that wraps your program. Here's what happens step by step:

**1. Wallet decrypts private account data locally**
Using your `nsk`, the wallet decrypts the ciphertext from the blockchain and reconstructs the plaintext account state. This never leaves your machine.

**2. Your program executes inside the zkVM**
The privacy circuit calls your program instruction with the decrypted data. Your program sees plain `AccountWithMetadata<T>` — no encryption, no special handling. It runs its logic exactly as in a public transaction.

**3. Privacy circuit validates execution**
The circuit checks:
- Balance conservation (total in = total out, no tokens created from nothing)
- Ownership rules (only the rightful owner signs for private accounts)
- Nullifier uniqueness (old commitments were valid and not already spent)

**4. ZK proof generated**
A single RISC Zero proof covers both your program logic AND the privacy circuit validation. The sequencer receives one proof that attests to everything.

**5. Submission**
The wallet submits: the proof, encrypted post-states (using recipient's `vpk`), new commitments, nullifiers, and view tags.

Your program code is unchanged. No privacy-specific code, no ZK primitives, no special return types.

## Building a Private Counter (Same Binary)

The Counter program from Chapter 4 already supports private transactions. No code changes needed.

To use it privately via the wallet API:

```rust
// Load the counter binary
let bytecode = std::fs::read("target/riscv-guest/release/counter")?;
let program = Program::new(bytecode)?;
let program_with_deps = ProgramWithDependencies::from(program);

// Instruction 0 = initialize, instruction 1 = increment
// For increment (index 1), no additional args
let instruction_data = risc0_zkvm::serde::to_vec(&(1u32,))?;

wallet.send_privacy_preserving_tx(
    vec![
        PrivacyPreservingAccount::PrivateOwned(counter_account_id),
        PrivacyPreservingAccount::PrivateOwned(caller_account_id),
    ],
    instruction_data,
    &program_with_deps,
).await?;
```

The counter increments privately. No one on-chain knows the account address, who called it, or what the new count is.

## Critical Notes

- **Private transfers generate ZK proofs locally** — this is CPU-intensive. Expect seconds to minutes depending on your hardware.
- **`lez-cli` cannot sign for private accounts** — use the wallet Rust API.
- **Same program binary** runs for both public and private transactions — no code changes needed.
- **Test with public first** — get your program logic correct with simple public transactions, then use it privately in production.
- **Sync before spend** — `wallet account sync-private` fetches fresh Merkle proofs. Stale proofs cause failed transactions.

> **💡 Tip:** The fact that the same binary works for both public and private txs has a profound implication: you test your program logic with simple public transactions, then deploy to production where users can choose to use it privately. No separate privacy testing surface.

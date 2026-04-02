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

## Public/Private Interoperability

This is one of LEZ's most important differentiators: **a single transaction can mix public and private accounts freely**. Private accounts can call public programs. Public accounts can interact with private accounts. A single instruction can operate on both simultaneously.

> **🔄 Coming from Solidity?** This doesn't exist anywhere in the EVM ecosystem. Zcash has two separate worlds — the shielded pool and the transparent pool — with a "shielding" operation to move between them. Tornado Cash is a mixer: you deposit publicly, withdraw privately through an intermediary. Neither can mix public and private state in a single atomic transaction. LEZ can.

Compare the models:

| System | Private ↔ Public interop |
|---|---|
| Zcash | Separate pools. "Shielding" tx to cross. |
| Tornado Cash | Deposit public → pool → withdraw private. No atomic mixing. |
| Aztec | Separate L2. Bridge required. |
| **LEZ** | **Single tx, mixed accounts. Atomic.** |

### A Real Mixed Example: DeFi Liquidity Pool

Consider a public liquidity pool program:
- The **pool account** is public — total liquidity, prices, and pool state are on-chain and visible to all
- A **depositor's wallet** is a private account — balance and identity are hidden

A single privacy-preserving transaction can:
1. Debit the depositor's private account (hidden amount)
2. Credit the public pool (visible pool state update)
3. Mint LP tokens to another private account (hidden balance)

What the world sees: the pool grew. By how much? Visible. Who deposited? Hidden. What they got back? Hidden.

```rust
// Mixed-visibility deposit
wallet.send_privacy_preserving_tx(
    vec![
        PrivacyPreservingAccount::Public(pool_account_id),        // pool visible
        PrivacyPreservingAccount::PrivateOwned(depositor_id),     // depositor hidden
        PrivacyPreservingAccount::PrivateForeign { npk, vpk },    // LP token recipient hidden
    ],
    deposit_instruction_data,
    &pool_program_with_deps,
).await?;
```

The program code doesn't change — it receives `AccountWithMetadata` for all accounts, public or private. The visibility is set by the wallet based on which accounts are private vs public.

### Why Per-Account Visibility Enables This

The visibility mask is set **per account**, not per transaction. This is what makes mixing possible.

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

`spel` **cannot sign for private accounts**. For custom programs that touch private accounts, you must use the wallet Rust API directly.

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
The privacy circuit calls your program instruction with the decrypted data. Your program sees plain `AccountWithMetadata` — no encryption, no special handling. It runs its logic exactly as in a public transaction.

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

## Privacy Security Model

Understanding exactly what is and isn't hidden:

| What | Visible? | Notes |
|---|---|---|
| Program being called | ✅ Always | `program_id` is always on-chain |
| Public account addresses | ✅ Always | Part of the transaction message |
| Public account state | ✅ Always | `public_post_states` in plaintext |
| Private account addresses | ❌ Hidden | Only commitments on-chain |
| Private account balances | ❌ Hidden | Encrypted in `encrypted_private_post_states` |
| Private account data | ❌ Hidden | Encrypted for recipient's `vpk` |
| Transfer amount (priv→priv) | ❌ Hidden | Inside the ZK proof |
| Transfer amount (pub→priv) | ❌ Hidden | The delta on the private side is hidden |
| Merkle tree root | ✅ Always | In `CommitmentSetDigest` — proves your commitment exists |
| Nullifier | ✅ Always | Proves you spent a commitment — but not which one |

The ZK proof attests to: "a valid commitment in the current tree was consumed, a valid new commitment was created, and the balance was conserved." The sequencer verifies the proof without learning which commitment was spent.

### What ZK Does NOT Hide

The **program being called** is always public. If you call a "private vote" program, everyone on-chain knows someone voted — just not who. This is a fundamental property of the account model: program execution is public metadata.

If you need to hide which program was called, that requires a different architecture entirely (and no blockchain does this today without significant tradeoffs).

> **⚠️ Warning:** The number of private accounts in a transaction may be visible from transaction size metadata. Be aware of correlation attacks based on transaction shape — two txs with the same structure may be linkable even if content is hidden.

## Troubleshooting Private Transactions

**"Proof generation failed"**
- Your program panicked inside the zkVM. Test with public accounts first using `spel` to validate the logic, then switch to private.

**"Nullifier already exists"**
- You tried to spend the same commitment twice. Run `wallet account sync-private` to refresh your local state — your wallet may have a stale view of which commitments you've spent.

**"Commitment not in tree"**
- You haven't run `wallet auth-transfer init` for this account, or you ran it on a different network than you're now querying. The commitment must exist in the tree before it can be spent.

**"Stale Merkle proof"**
- The commitment tree has advanced since you last synced. Run `wallet account sync-private` and retry.

**"Cannot sign for private account"**
- You're using `spel` to call an instruction that touches a private account. Switch to the wallet Rust API. `spel` can only sign for public accounts.

**"Invalid balance conservation"**
- Your program's logic doesn't preserve total balance across all accounts. The privacy circuit checks that `sum(inputs) == sum(outputs)`. Fees, if any, are deducted from balance explicitly.

## Critical Notes

- **Private transfers generate ZK proofs locally** — this is CPU-intensive. Expect seconds to minutes depending on your hardware.
- **`spel` cannot sign for private accounts** — use the wallet Rust API.
- **Same program binary** runs for both public and private transactions — no code changes needed.
- **Test with public first** — get your program logic correct with simple public transactions, then use it privately in production.
- **Sync before spend** — `wallet account sync-private` fetches fresh Merkle proofs. Stale proofs cause failed transactions.

> **💡 Tip:** The fact that the same binary works for both public and private txs has a profound implication: you test your program logic with simple public transactions, then deploy to production where users can choose to use it privately. No separate privacy testing surface.

> **💡 Tip:** Share only your `npk` and `vpk` to receive private funds. Keep your `nsk` offline — it's the key that can spend all your private commitments. There is no recovery mechanism if `nsk` is lost.

## Designing Programs for Privacy

When writing programs that will be used with private accounts, a few design considerations:

### Avoid On-Chain Leakage Through Public Side-Channels

If your program emits an event log or updates a public counter every time a private transfer occurs, you've leaked timing metadata. Anyone watching the public counter can infer when private activity happened. Prefer designs where private-only paths have **no public state changes** unless that's intentional (e.g., a public liquidity pool where the deposit amount is private but the pool size is public by design).

### Program Logic Must Be Balance-Conserving

The privacy circuit enforces `sum(input balances) == sum(output balances)` across all accounts in the transaction. Your program logic must not create or destroy balance — only move it. If your program charges a fee, it must explicitly credit a fee account:

```rust
// ❌ Wrong — balance disappears into the void
let sender_new_balance = sender.balance() - amount - fee;
let receiver_new_balance = receiver.balance() + amount;

// ✅ Correct — fee goes somewhere explicit
let sender_new_balance = sender.balance() - amount - fee;
let receiver_new_balance = receiver.balance() + amount;
let fee_collector_new_balance = fee_collector.balance() + fee;
// All three accounts must be in post_states
```

### Keep State Minimal in Private Accounts

Private account data is encrypted and stored on-chain. Every byte costs. Keep your state struct small. Large `Vec<T>` fields in private account state are expensive — prefer splitting complex state across multiple accounts with PDAs.

### The `data` Field in Commitments

The commitment formula includes `SHA256(data)` — the hash of your account's data field. This means:
- Changing any field in your state struct changes the commitment
- The commitment hides the actual data, but attests to it cryptographically
- Your program logic in the zkVM sees the plaintext `data` — it's decrypted locally before the proof is generated

This design means your program code doesn't need to know anything about encryption. It operates on plaintext. The privacy circuit handles the encryption/decryption boundary.

> **🔄 Coming from Solidity?** In Solidity, writing a privacy-preserving contract requires either a full ZK framework (Aztec, circom), a mixer contract (Tornado Cash), or a trusted enclave. In LEZ, privacy is the protocol's job. Your program is plain Rust with no ZK primitives. The privacy circuit wraps it at the protocol level — you opt into privacy at the call site, not in the contract code.

> **💡 Tip:** Use small, focused state structs in private accounts. Every unnecessary field adds to the commitment size and proof generation time. When in doubt, split state across two accounts using PDAs — one for public metadata, one for private balances.

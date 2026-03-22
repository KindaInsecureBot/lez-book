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

Private accounts use a derived key hierarchy:

```
nsk (nullifier secret key)
  └─ npk = SHA256(nsk)  (nullifier public key)
       └─ vpk = secp256k1_pubkey(nsk)  (viewing public key)
            └─ ssk = ephemeral signing key
```

- **nsk**: your private key for nullifier derivation. Never share this.
- **npk**: your public key for commitments. Others use this to send you private funds.
- **vpk**: for view-key sharing — lets someone scan the chain to see your private txs without spending them
- **ssk**: ephemeral signing key for each transaction

## Commitments and Nullifiers

Private accounts use a Zcash-style commitment/nullifier scheme:

**Commitment** (what goes on-chain when you receive):
```
commitment = SHA256(npk || program_owner || balance || nonce || SHA256(data))
```

**Nullifier** (what goes on-chain when you spend):
```
nullifier = f(nsk, old_commitment)
```

The commitment hides all the account details. The nullifier prevents double-spending. The link between commitment and nullifier is only knowable with `nsk`.

## View Tags

View tags are a 1-byte filter on each private transaction that lets your wallet efficiently scan for relevant transactions:

```
view_tag = first_byte(ECDH_shared_secret(sender_ephemeral_key, recipient_vpk))
```

Your wallet computes the view tag for each transaction and compares. If it doesn't match (99.6% of transactions), skip. If it matches, do the full decryption to check if it's yours.

This gives efficient wallet scanning without revealing which transactions are yours to observers.

## Visibility Mask

Each account in a private transaction has a visibility level:

```rust
pub enum Visibility {
    Public = 0,        // plaintext on chain
    PrivateAuth = 1,   // encrypted, owner can decrypt
    PrivateUnauth = 2, // encrypted, no one can decrypt without view key
}
```

## Wallet Operations for Private Accounts

```bash
# Create a new private account
wallet account new private

# Initialize a private program account
wallet auth-transfer init --program <program-id> --account <addr>

# Sync your private accounts (scan chain for incoming txs)
wallet account sync-private

# Send a private transfer
wallet auth-transfer send --to <npk> --amount 100 --program <program-id>
```

## Using Custom Programs with Private Transactions

The wallet's `send_privacy_preserving_tx()` method accepts any SPEL program:

```rust
// Rust client code for privacy-preserving tx
wallet.send_privacy_preserving_tx(
    program_id,
    instruction_name,
    accounts,
    args,
    visibility_mask,
).await?;
```

The same program binary (e.g., your Counter) runs inside the privacy circuit. Your instruction code doesn't change.

## What the Privacy Circuit Does

The privacy circuit:
1. Decrypts the private input accounts
2. Runs your program instruction on them
3. Re-encrypts the output accounts
4. Generates commitment/nullifier pairs
5. Wraps everything in a ZK proof

Your program code never sees the encryption — it just sees accounts with data, runs its logic, and returns updated accounts. The privacy layer handles the rest.

## Building a Private Counter (Same Binary)

The Counter program from Chapter 4 already supports private transactions. To use it privately:

```bash
# Initialize a private counter account
wallet auth-transfer init \
  --program COUNTER_PROGRAM_ID \
  --instruction initialize \
  --account <private-counter-addr> \
  --visibility private-auth

# Increment it privately
wallet auth-transfer send \
  --program COUNTER_PROGRAM_ID \
  --instruction increment \
  --account <private-counter-addr>
```

No changes to the program code.

> **💡 Tip:** The fact that the same binary works for both public and private txs has a profound implication: you test your program logic with simple public transactions, then deploy to production where users can choose to use it privately. No separate privacy testing surface.

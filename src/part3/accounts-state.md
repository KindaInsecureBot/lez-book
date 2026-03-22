# Accounts & State

This chapter covers the biggest mental shift when moving to LEZ: state does not live inside your program.

## The Core Insight

In Ethereum, state lives INSIDE the contract. The contract's storage is part of the contract.

In LEZ (like Solana), state lives in ACCOUNTS that are OWNED BY a program. The program itself is stateless code.

> **🔄 Coming from Solidity?** Imagine if every Solidity state variable was in a separate struct, stored in a separate address on-chain, and your contract just processed it when called. That's the LEZ account model. Your program is the logic; accounts are the database.

## Account Structure

Every account has:

```rust
pub struct Account {
    pub id: [u8; 32],            // address
    pub data: Vec<u8>,           // serialized program data
    pub balance: u64,            // native token balance
    pub nonce: u64,              // replay protection
    pub program_owner: [u8; 32], // which program "owns" this account
}
```

The `program_owner` field is critical — it says which program is allowed to modify this account's data.

## Account Ownership and Balance Rules

The protocol enforces one critical ownership rule:

> **Only the `program_owner` program can decrease an account's balance.**

Any program can read any account. Any program can increase an account's balance (deposit). But only the owning program can decrease it (withdraw). This is enforced by the runtime, not by your code.

This means you can safely accept deposits from anyone without worrying about unauthorized withdrawals.

## AccountWithMetadata

In SPEL, accounts are typed as `AccountWithMetadata` — there is **no generic type parameter**. Account data is raw bytes, accessed via `.account.data` and deserialized manually:

```rust
// All accounts are AccountWithMetadata — no generic param
let counter: AccountWithMetadata;

// Access raw account data (Vec<u8>)
let raw_data: &[u8] = &counter.account.data;

// Get the account ID (address) — it's inside the inner Account struct
let addr: [u8; 32] = counter.account.id;

// Deserialize manually (borsh_derive doesn't compile for riscv32im)
let state = CounterState::from_bytes(raw_data)
    .ok_or(LezError::DeserializationError { account_index: 0, message: "bad data".into() })?;

// Update: clone the inner Account and set new data
let mut updated = counter.account.clone();
updated.data = new_state.to_bytes().try_into().unwrap();
// Then pass `updated` (an Account) to AccountPostState::new(updated)
```

## AccountPostState — new vs new_claimed

When returning accounts from an instruction, you must choose:

```rust
// For NEW accounts being initialized for the first time
AccountPostState::new_claimed(account)

// For EXISTING accounts being updated
AccountPostState::new(account)
```

The `new_claimed` variant tells the sequencer: "this account didn't exist before, create it now with this data." The `new` variant updates an existing account.

### Conditional Init: new_claimed_if_default

A common pattern is "initialize if not yet initialized, otherwise update." Rather than two separate instructions, you can use a conditional approach:

```rust
#[instruction]
pub fn ensure_initialized(
    #[account(mut)] account: AccountWithMetadata,
    #[account(signer)] owner: AccountWithMetadata,
) -> LezResult {
    let post_state = if account.account.data.is_empty() {
        // Account doesn't exist yet — initialize and claim it
        let state = MyState { owner: owner.account.id, value: 0 };
        let mut new_account = account.account.clone();
        new_account.data = state.to_bytes().try_into().unwrap();
        AccountPostState::new_claimed(new_account)
    } else {
        // Account already exists — pass through unchanged
        AccountPostState::new(account.account.clone())
    };

    Ok(LezOutput::states_only(vec![
        post_state,
        AccountPostState::new(owner.account.clone()),
    ]))
}
```

This lets you write idempotent instructions that work both on first call and subsequent calls.

> **⚠️ Warning:** You MUST return ALL accounts in `post_states` — even accounts you didn't modify. The sequencer uses the post_states list as the complete new state of all accounts involved in the transaction. If you omit an account, its state is NOT preserved.

## Must Return ALL Accounts

This is a common source of bugs:

```rust
// ❌ Wrong — forgot to include authority in post_states
Ok(LezOutput::states_only(vec![
    AccountPostState::new(counter.account.clone()),
]))

// ✅ Correct — all accounts returned
Ok(LezOutput::states_only(vec![
    AccountPostState::new(counter.account.clone()),
    AccountPostState::new(authority.account.clone()),
]))
```

## Manual Serialization (Critical for Guest Code)

> **⚠️ Warning:** `borsh_derive` proc macros (`#[derive(BorshSerialize, BorshDeserialize)]`) **do not compile** for the `riscv32im` guest target. This is a known limitation. If you try to derive these traits in your guest crate, compilation will fail.

This is a major practical issue. The solution is **manual serialization**:

```rust
// ❌ WRONG — borsh_derive won't compile in riscv32im guest
#[derive(BorshSerialize, BorshDeserialize)]
pub struct MyState {
    pub owner: [u8; 32],
    pub value: u64,
}

// ✅ RIGHT — manual serialization, works everywhere
pub struct MyState {
    pub owner: [u8; 32],
    pub value: u64,
}

impl MyState {
    pub const SIZE: usize = 32 + 8; // owner ([u8; 32]) + value (u64)

    pub fn to_bytes(&self) -> Vec<u8> {
        let mut buf = Vec::with_capacity(Self::SIZE);
        buf.extend_from_slice(&self.owner);
        buf.extend_from_slice(&self.value.to_le_bytes());
        buf
    }

    pub fn from_bytes(data: &[u8]) -> Option<Self> {
        if data.len() < Self::SIZE {
            return None;
        }
        let owner: [u8; 32] = data[..32].try_into().ok()?;
        let value = u64::from_le_bytes(data[32..40].try_into().ok()?);
        Some(Self { owner, value })
    }
}
```

### The Core Crate Pattern

The standard project layout puts shared types in a `<name>_core` crate (e.g., `counter_core`) used by both the guest program and the IDL generator:

```
counter/
├── counter_core/   # Shared types — compiled for both native and riscv32im
│   └── src/lib.rs  # CounterState, manual serialization, constants
├── methods/        # RISC Zero guest crate — compiled only for riscv32im
│   └── guest/src/bin/counter.rs  # Uses counter_core types
└── examples/       # Host-side binaries — compiled only for native
    └── src/bin/
        ├── generate_idl.rs  # Uses counter_core types
        └── counter_cli.rs
```

The core crate CAN derive `serde::Serialize`/`Deserialize` — those work fine everywhere. Only `borsh_derive` proc macros are broken for `riscv32im`. Manual Borsh serialization in the core crate works in both targets.

```rust
// core/src/lib.rs — shared between guest and host

use serde::{Serialize, Deserialize};

// serde derives work fine (used for IDL generation)
#[derive(Serialize, Deserialize, Clone, Debug, Default)]
pub struct MyState {
    pub owner: [u8; 32],
    pub value: u64,
}

impl MyState {
    pub const SIZE: usize = 32 + 8;

    // Manual Borsh serialization — works in riscv32im guest
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut buf = Vec::with_capacity(Self::SIZE);
        buf.extend_from_slice(&self.owner);
        buf.extend_from_slice(&self.value.to_le_bytes());
        buf
    }

    pub fn from_bytes(data: &[u8]) -> Option<Self> {
        if data.len() < Self::SIZE { return None; }
        let owner: [u8; 32] = data[..32].try_into().ok()?;
        let value = u64::from_le_bytes(data[32..40].try_into().ok()?);
        Some(Self { owner, value })
    }
}
```

## Account Ownership Model

Only the `program_owner` program can modify an account's data. This is enforced by the protocol.

Use `#[account(owner = PROGRAM_ID)]` to assert that an account is owned by your program:

```rust
#[instruction]
pub fn transfer(
    #[account(mut, owner = COUNTER_PROGRAM_ID)] counter: AccountWithMetadata,
    // ...
)
```

> **💡 Tip:** The `owner` constraint is your first line of defense. Always constrain the owner of accounts you write to. Without it, someone could pass an account owned by a different program with maliciously crafted data.

## Solidity vs LEZ State Pattern

**Solidity:**

```solidity
contract Token {
    mapping(address => uint256) public balances;  // state lives here

    function transfer(address to, uint256 amount) external {
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

**LEZ:**

```rust
// State lives in separate accounts, not in the program
#[instruction]
pub fn transfer(
    #[account(mut)] sender_account: AccountWithMetadata,
    #[account(mut)] receiver_account: AccountWithMetadata,
    #[account(signer)] sender: AccountWithMetadata,
    amount: u64,
) -> LezResult {
    // Deserialize both accounts manually from raw bytes
    let mut sender_state = TokenBalance::from_bytes(&sender_account.account.data).unwrap();
    let mut receiver_state = TokenBalance::from_bytes(&receiver_account.account.data).unwrap();

    // Apply transfer
    sender_state.balance -= amount;
    receiver_state.balance += amount;

    // Write back
    let mut updated_sender = sender_account.account.clone();
    updated_sender.data = sender_state.to_bytes().try_into().unwrap();
    let mut updated_receiver = receiver_account.account.clone();
    updated_receiver.data = receiver_state.to_bytes().try_into().unwrap();

    // All three accounts must be returned
    Ok(LezOutput::states_only(vec![
        AccountPostState::new(updated_sender),
        AccountPostState::new(updated_receiver),
        AccountPostState::new(sender.account.clone()),
    ]))
}
```

## Vec<AccountWithMetadata> — Variable-Length Accounts

For instructions that need a variable number of accounts:

```rust
#[instruction]
pub fn batch_operation(
    accounts: Vec<AccountWithMetadata>,
    #[account(signer)] authority: AccountWithMetadata,
) -> LezResult {
    // Process each account — pass through as-is (or modify and write back)
    let mut post_states: Vec<AccountPostState> = accounts.into_iter()
        .map(|a| AccountPostState::new(a.account))
        .collect();
    post_states.push(AccountPostState::new(authority.account.clone()));
    Ok(LezOutput::states_only(post_states))
}
```

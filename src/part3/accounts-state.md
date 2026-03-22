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

In SPEL, you work with `AccountWithMetadata<T>` where `T` is your state struct:

```rust
// An account holding CounterState
let counter: AccountWithMetadata<CounterState>;

// Access the data
let state: &CounterState = counter.data();

// Get the account ID (address)
let addr: &[u8; 32] = counter.id();

// Create a new version with updated data
let updated = counter.with_data(new_state);
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
    #[account(mut)] account: AccountWithMetadata<MyState>,
    #[account(signer)] owner: AccountWithMetadata<()>,
) -> Result<LezOutput, LezError> {
    let post_state = if *account.raw_account() == Account::default() {
        // Account doesn't exist yet — claim it
        let state = MyState { owner: *owner.id(), value: 0 };
        AccountPostState::new_claimed(account.with_data(state))
    } else {
        // Account already exists — just update
        AccountPostState::new(account)
    };

    Ok(LezOutput::states_only(vec![
        post_state,
        AccountPostState::new(owner),
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
    AccountPostState::new(counter),
]))

// ✅ Correct — all accounts returned
Ok(LezOutput::states_only(vec![
    AccountPostState::new(counter),
    AccountPostState::new(authority),
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

The standard project layout puts shared types in a `core` crate that is used by both the guest (program) and host (client, IDL gen):

```
my_program/
├── core/           # Shared types — compiled for both native and riscv32im
│   └── src/lib.rs  # MyState, manual serialization, constants
├── guest/          # Program binary — compiled only for riscv32im
│   └── src/main.rs # Uses core types
├── host/           # Client code — compiled only for native
│   └── src/main.rs # Uses core types
└── idl-gen/        # IDL generator — compiled only for native
    └── src/main.rs
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
    #[account(mut, owner = COUNTER_PROGRAM_ID)] counter: AccountWithMetadata<CounterState>,
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
    #[account(mut)] sender_account: AccountWithMetadata<TokenBalance>,
    #[account(mut)] receiver_account: AccountWithMetadata<TokenBalance>,
    #[account(signer)] sender: AccountWithMetadata<()>,
    amount: u64,
) -> Result<LezOutput, LezError> {
    // Both accounts explicitly passed in
    // Both must be returned in post_states
}
```

## Vec<AccountWithMetadata> — Variable-Length Accounts

For instructions that need a variable number of accounts:

```rust
#[instruction]
pub fn batch_operation(
    accounts: Vec<AccountWithMetadata<MyState>>,
    #[account(signer)] authority: AccountWithMetadata<()>,
) -> Result<LezOutput, LezError> {
    // Process each account
    let mut post_states: Vec<AccountPostState> = accounts.into_iter()
        .map(|a| AccountPostState::new(a))
        .collect();
    post_states.push(AccountPostState::new(authority));
    Ok(LezOutput::states_only(post_states))
}
```

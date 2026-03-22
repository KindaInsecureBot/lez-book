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
    pub data: Vec<u8>,           // borsh-serialized program data
    pub balance: u64,            // native token balance
    pub nonce: u64,              // replay protection
    pub program_owner: [u8; 32], // which program "owns" this account
}
```

The `program_owner` field is critical — it says which program is allowed to modify this account's data.

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

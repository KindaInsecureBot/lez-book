# Gotchas & Troubleshooting

> Things that WILL bite you.

This chapter is a field guide to the bugs and surprises that have caught real developers building on LEZ. Read it before you start, and return to it when something inexplicably breaks.

---

### 1. Genesis Accounts Must Sign (NSSA Rule 7)

> **⚠️ Warning:** Derived accounts (PDAs) cannot be signers. Only genesis accounts (accounts created at chain genesis, not derived from seeds) can sign transactions. This is NSSA rule 7.

**Symptom**: Transaction fails with "invalid signer" even though the account exists.

**Cause**: You're trying to use a PDA-derived account as a signer.

**Fix**: Use a genesis account as the signer. PDAs are for state, not signing.

```rust
// ❌ Wrong — PDA can't sign
#[account(signer, pda = account("user"))] user_pda: AccountWithMetadata<UserState>

// ✅ Correct — genesis account signs, PDA is the state
#[account(signer)] user: AccountWithMetadata<()>,
#[account(mut, pda = account("user"))] user_state: AccountWithMetadata<UserState>,
```

> **🔄 Coming from Solidity?** In Solidity, `msg.sender` is always an EOA or contract. In LEZ, only genesis accounts can be `msg.sender` equivalents. PDAs are state containers, not actors.

---

### 2. borsh_derive Doesn't Compile for riscv32im

> **⚠️ Warning:** The `borsh_derive` crate DOES NOT compile for the `riscv32im` RISC Zero guest target. Using `borsh_derive` as a separate dependency will cause cryptic compilation errors.

**Symptom**: Build fails with errors referencing proc-macro compilation for riscv32im.

**Cause**: `borsh_derive` uses proc-macros that don't cross-compile to the guest target.

**Fix**: Use `borsh = { version = "1", features = ["derive"] }` — this uses borsh's own built-in derive, which works correctly.

```toml
# ❌ Wrong
[dependencies]
borsh = "1"
borsh_derive = "1"   # <-- this will break the guest build

# ✅ Correct
[dependencies]
borsh = { version = "1", features = ["derive"] }
```

---

### 3. lez-cli pda Bug with Typed Seeds

> **⚠️ Warning:** `lez-cli pda` gives WRONG addresses when seeds contain `u64` or `u128` typed arguments. The CLI's serialization doesn't match the on-chain SHA-256 derivation.

**Symptom**: The PDA address printed by `lez-cli pda` doesn't match the actual on-chain address.

**Cause**: Type encoding mismatch between the CLI and the SPEL macro for numeric seed types.

**Fix**: For numeric seeds, compute PDAs manually or derive them from transaction results. Literal and account seeds work correctly.

```bash
# ❌ Don't rely on this for numeric seeds
lez-cli pda --program <id> --seeds tile 10 20

# ✅ Instead: read the account address from the transaction receipt
# or compute SHA256(program_id || "tile" || x_le_bytes || y_le_bytes) manually
```

---

### 4. RocksDB Stale State

> **⚠️ Warning:** When you upgrade or switch between lssa versions, the old RocksDB state database persists. This causes failures that look like network or program errors but are actually state corruption.

**Symptom**: Sequencer starts but fails on seemingly valid transactions; account states look wrong.

**Cause**: The RocksDB schema changed between lssa versions.

**Fix**:

```bash
# Find the rocksdb directory (default location) and wipe it
rm -rf ~/.lssa/db

# Restart sequencer fresh
sequencer_runner --port 3040
```

> **💡 Tip:** Any time you see "this worked yesterday and I changed nothing," suspect stale RocksDB state. Wiping the DB is always safe in local development — you'll just lose any test state you haven't checkpointed.

---

### 5. lssa Version Pinning

> **⚠️ Warning:** lez-cli pins to a specific lssa revision for RPC compatibility. Using the wrong lssa version causes subtle failures, not obvious error messages.

**Symptom**: Transactions seem to submit but never appear; RPC calls fail with cryptic errors.

**Cause**: lssa rev `767b5af` has a broken commitment RPC. Other version mismatches cause similar issues.

**Fix**: Use the `main` branch of lssa. Check lez-cli's source for the pinned rev it expects, and use that.

```bash
# Check what rev lez-cli expects
grep -r "lssa" spel/lez-cli/Cargo.toml
```

---

### 6. init Implies mut

> **⚠️ Warning:** `#[account(init)]` already implies mutability. Don't write `#[account(init, mut)]` — this is redundant at best and may cause macro errors.

**Fix**: Use `#[account(init)]` alone.

```rust
// ❌ Wrong
#[account(init, mut)] new_account: AccountWithMetadata<MyState>

// ✅ Correct
#[account(init)] new_account: AccountWithMetadata<MyState>
```

---

### 7. All Accounts Must Be in post_states

> **⚠️ Warning:** Every account passed to your instruction MUST be returned in `post_states`, even if you didn't modify it. Omitting an account doesn't preserve its state — it's treated as if it was deleted.

**Symptom**: Account balance or data "disappears" after calling an instruction.

**Cause**: The account was passed in but not returned in `post_states`.

**Fix**: Always collect all input accounts into your return vec:

```rust
// Safe pattern: collect all accounts, modify as needed
let mut post_states = vec![];
post_states.push(AccountPostState::new(counter.with_data(new_state)));
post_states.push(AccountPostState::new(authority)); // don't forget!
Ok(LezOutput::states_only(post_states))
```

> **🔄 Coming from Solidity?** In Solidity, you modify storage in-place and don't return anything. In LEZ, your instruction is a pure function: the sequencer replaces account states with exactly what you return. If you don't return an account, it's gone.

---

### 8. Account Ordering in Function Signatures

> **⚠️ Warning:** Accounts MUST come before scalar arguments in instruction function signatures. Mixing them causes macro compilation errors.

```rust
// ❌ Wrong — scalar arg mixed in before all accounts
pub fn my_instruction(
    #[account(signer)] user: AccountWithMetadata<()>,
    value: u64,                                          // <-- too early
    #[account(mut)] state: AccountWithMetadata<MyState>,
) -> ...

// ✅ Correct — all accounts first, then scalars
pub fn my_instruction(
    #[account(signer)] user: AccountWithMetadata<()>,
    #[account(mut)] state: AccountWithMetadata<MyState>,
    value: u64,
) -> ...
```

---

### 9. OOM During Build

> **⚠️ Warning:** RISC Zero guest compilation uses a lot of memory. On machines with less than 8 GB RAM or in constrained containers, the build will silently OOM and produce corrupted binaries or just fail.

**Fix**:

```bash
cargo build --release --jobs 2
# or in Makefile:
CARGO_BUILD_JOBS=2 cargo build --release
```

Limiting parallelism keeps peak memory manageable. Two jobs is a safe default for CI environments with 4 GB RAM.

---

### 10. Rest Account CLI Flag

> **⚠️ Warning:** When calling instructions with `Vec<AccountWithMetadata<T>>` (rest accounts), the CLI flag is `--<name>` (e.g., `--tiles`), NOT `--<name>-account`. Check your IDL to see the exact flag name.

```bash
# ❌ Wrong
lez-cli call --instruction batch_update --tiles-account <addr>

# ✅ Correct (check your IDL for the actual field name)
lez-cli call --instruction batch_update --tiles <addr>
```

---

### 11. Scaffold Missing risc0 Metadata

> **⚠️ Warning:** `lez-cli init` scaffold may be missing `risc0-zkvm` metadata in `methods/Cargo.toml`. This causes build failures when the zkVM tries to locate guest binary metadata.

**Fix**: Manually add the required risc0 metadata to `methods/Cargo.toml`:

```toml
[package.metadata.risc0]
methods = ["guest"]
```

Without this, `build.rs` in the methods crate cannot locate the guest binary and the build silently produces no ELF file.

---

### 12. Determinism Violations

> **⚠️ Warning:** Everything in your program must be deterministic. The ZK proof system requires that re-executing the guest binary with the same inputs always produces the same output. Any non-determinism invalidates the proof.

Non-deterministic operations that will break your program:

- `std::time::SystemTime::now()` — wall clock time
- `rand::random()` — non-seeded random numbers
- HashMap iteration order (use BTreeMap instead)
- Floating-point operations (results can differ between hardware)

```rust
// ❌ Wrong — non-deterministic
let now = std::time::SystemTime::now();

// ✅ Correct — use a block timestamp from your program inputs
// (pass it in as an argument or read from a known account)
let timestamp: u64 = clock_account.data().timestamp;
```

---

### 13. Private Account Scan Required After Receive

> **⚠️ Warning:** Private accounts sent to you are NOT automatically visible in your wallet. You must run `wallet account sync-private` to scan the chain and decrypt incoming private transactions.

**Symptom**: Someone sent you tokens or a private message but your wallet shows nothing.

**Fix**:

```bash
wallet account sync-private
```

This scans recent blocks for commitments whose view tags match your vpk, decrypts them using your nsk, and adds them to your local wallet.

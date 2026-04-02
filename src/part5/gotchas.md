# Gotchas & Troubleshooting

> Things that WILL bite you.

This chapter is a field guide to the bugs and surprises that have caught real developers building on LEZ. Read it before you start, and return to it when something inexplicably breaks.

---

## 1. Genesis Accounts MUST Be Signers — Not PDAs

**Explanation:** In LEZ, only genesis accounts (keypair-backed accounts created via `wallet account new`) can sign transactions. PDAs (Program Derived Addresses) are derived from seeds and have no associated private key, so they cannot produce signatures. The sequencer enforces this at validation time.

**Symptoms:**
- Transaction rejected at sequencer with a signature validation error
- Or, more confusingly, the instruction appears to execute but the signer constraint is silently bypassed, leading to authorization bugs

**Fix:** Always pass genesis accounts (wallet keypair accounts) where `#[account(signer)]` is expected. Never pass a PDA as a signer. If your design requires a PDA to "authorize" something, restructure: have the genesis account sign and validate ownership through a PDA seed relationship instead.

```rust
// ✅ Correct: genesis account signs
#[instruction]
pub fn transfer(
    authority: AccountWithMetadata,  // #[account(signer)] — must be genesis
    vault: AccountWithMetadata,      // PDA — does NOT sign
    amount: u64,
) -> LezResult { ... }
```

> **⚠️ Warning:** The sequencer may not always produce a clear error message when a PDA is passed as a signer. If you see commitment mismatch or authorization errors with no obvious cause, check that your signer account is a genesis account.

> **🔄 Coming from Solidity?** This is similar to the distinction between an EOA (Externally Owned Account) and a contract: contracts cannot sign EIP-712 messages directly (without ERC-1271). PDAs are the LEZ equivalent of contract accounts — they can hold state and be owned by a program, but cannot produce keypair signatures.

---

## 2. `borsh_derive` Does Not Compile for `riscv32im`

**Explanation:** The `borsh_derive` crate provides procedural macros (`#[derive(BorshSerialize, BorshDeserialize)]`). Proc-macros run on the host compiler, but when building the RISC Zero guest binary for the `riscv32im-risc0-zkvm-elf` target, the build system attempts to compile proc-macro dependencies for the guest target — which has no `proc_macro` support.

**Symptoms:**
```
error[E0463]: can't find crate for `proc_macro`
  --> ~/.cargo/registry/src/.../borsh-derive-1.x.x/src/lib.rs:1:1
```
The build fails entirely. This error often appears deep in a dependency chain, making it hard to trace back to your code.

**Fix:** Do not use `#[derive(BorshSerialize, BorshDeserialize)]` in guest code. Instead, implement borsh serialization manually:

```rust
// ❌ Broken in guest: proc-macro won't compile
#[derive(BorshSerialize, BorshDeserialize)]
pub struct MyState {
    pub value: u64,
    pub owner: [u8; 32],
}

// ✅ Correct: manual impl
impl BorshSerialize for MyState {
    fn serialize<W: borsh::io::Write>(&self, writer: &mut W) -> borsh::io::Result<()> {
        self.value.serialize(writer)?;
        self.owner.serialize(writer)?;
        Ok(())
    }
}

impl BorshDeserialize for MyState {
    fn deserialize_reader<R: borsh::io::Read>(reader: &mut R) -> borsh::io::Result<Self> {
        Ok(Self {
            value: BorshDeserialize::deserialize_reader(reader)?,
            owner: BorshDeserialize::deserialize_reader(reader)?,
        })
    }
}
```

> **💡 Tip:** Put your state struct definitions in a shared `types` crate compiled separately for host and guest. The guest crate imports the types but only with manual borsh impls; the host crate can use derive macros freely.

> **⚠️ Warning:** This also applies to any other proc-macro crates you might pull in transitively (`serde_derive`, `thiserror`, etc.). Audit your guest `Cargo.toml` for proc-macro dependencies before debugging build failures. Run `cargo tree --target riscv32im-risc0-zkvm-elf` to inspect the dependency graph.

---

## 3. `spel pda` Gives Wrong Addresses for u64/u128 Seeds

**Explanation:** The `spel pda` command has a known bug where integer seeds (u64, u128) are not serialized correctly when computing PDA addresses. The CLI produces an address that does not match what the sequencer assigns during an `init` call. String seeds work correctly.

**Symptoms:**
- PDA address from `spel pda` doesn't match the address shown in sequencer logs after initialization
- Subsequent calls using the CLI-derived address fail with `InvalidAccountOwner` or account-not-found errors
- Everything looks right on the client side but the sequencer rejects the account

**Fix:** For programs that use integer seeds, get the canonical PDA address from the sequencer logs during the first successful `init` call, not from `spel pda`:

```bash
# Initialize and capture the assigned PDA address from sequencer output
make cli ARGS="initialize --authority-account <genesis-addr>"
# Look in sequencer logs for: "Created account: <pda-addr>"
# Use that address for all subsequent calls
```

If you need to compute PDA addresses programmatically (e.g., in a client), implement the seed hashing logic directly using the same algorithm as the sequencer, rather than relying on the CLI.

> **💡 Tip:** String seeds are a reliable workaround. If your seed is an integer `user_id: u64`, consider encoding it as a decimal string: `format!("user:{}", user_id)`.

> **⚠️ Warning:** This is a CLI-only bug. The on-chain PDA derivation is correct. Do not change your program logic — only change how you compute the address on the client side.

---

## 4. RocksDB Stale State Persists Across `lssa` Restarts

**Explanation:** `lssa` uses RocksDB as its persistent state store. When you restart `lssa` — especially when switching between versions or after a crash — the old database state is still on disk. If the new `lssa` version has a different state schema, or if you want a clean slate for testing, stale RocksDB state causes subtle failures.

**Symptoms:**
- `lssa` starts without error but RPC calls return unexpected results
- Commitment queries return data that should have been cleared
- "Commitment mismatch" errors on the first transaction after restart
- Errors that look like valid RPC responses but with wrong content (correct JSON shape, wrong values)

**Fix:** Run `just clean` from the `~/lez` directory, then restart:

```bash
cd ~/lez
# Stop the sequencer first, then:
just clean

# Then restart
just run-sequencer
```

`just clean` removes `sequencer/service/rocksdb`, `indexer/service/rocksdb`, `sequencer/service/bedrock_signing_key`, and `wallet/configs/debug/storage.json` in one step. You can also delete just the RocksDB directories manually if you want to preserve wallet storage:

```bash
rm -rf ~/lez/sequencer/service/rocksdb
rm -rf ~/lez/indexer/service/rocksdb
```

> **⚠️ Warning:** This deletes all sequencer state, including all accounts and commitments. On devnet this is fine. On any environment with real state, take a backup first.

---

## 5. `lssa` Revision `767b5af` Has Broken Commitment RPC

**Explanation:** A specific commit of `lssa` (`767b5af`) introduced a regression where commitment queries via the RPC API return empty results regardless of actual stored state. The program logic and sequencing still work, but anything that queries commitments (wallet scanning, client-side state reads) returns nothing.

**Symptoms:**
- `wallet account inspect <pda-addr>` shows no data even after successful transactions
- `wallet account sync-private` finds no commitments
- Client calls that read account state return empty or zeroed data
- No errors are reported — responses are well-formed but empty

**Fix:** Use the `main` branch of `lssa` rather than pinning to `767b5af`. If you need to pin a version for reproducibility, pin to a commit after this regression was fixed:

```bash
# In your logos-execution-zone build/install process, use main or a known-good commit
git clone https://github.com/logos-blockchain/logos-execution-zone.git ~/lez
cd ~/lez
git checkout main   # not 767b5af
cargo build --release
```

> **⚠️ Warning:** This bug is particularly insidious because it looks like a client-side issue (your wallet or client code can't read state) when the problem is the sequencer's RPC layer. If wallet scanning finds nothing after confirmed transactions, suspect this first.

---

## 6. `init` Implies `mut` — Don't Combine Them

**Explanation:** In LEZ's account constraint system, `#[account(init)]` already implies that the account will be created and written — it is implicitly mutable. Adding `mut` alongside `init` is redundant and can cause constraint conflicts or unexpected behavior.

**Symptoms:**
- Macro expansion errors or constraint validation failures at compile time
- Runtime errors about conflicting account constraints
- Confusing error messages that reference the account declaration line

**Fix:** Use only `init` without `mut`:

```rust
// ❌ Redundant and potentially broken
#[account(init, mut, seeds = [b"state", authority.key().as_ref()])]
pub state: AccountWithMetadata,

// ✅ Correct
#[account(init, seeds = [b"state", authority.key().as_ref()])]
pub state: AccountWithMetadata,
```

> **💡 Tip:** Only use `mut` on accounts that already exist and are being modified. Use `init` only for accounts being created for the first time. The two are mutually exclusive from a semantic standpoint.

---

## 7. ALL Accounts Must Be Returned in `post_states`

**Explanation:** The LEZ sequencer performs account balancing: the number of accounts returned in `LezOutput` must exactly match the number of accounts passed in. Even accounts you did not read from or modify must be included in `post_states`. The sequencer does not infer which accounts were touched.

**Symptoms:**
```
Error: account count mismatch: expected 3, got 2
```
- Transaction is rejected at the sequencer
- The error clearly states a count mismatch, but it's easy to miss which account was dropped

**Fix:** Return every received account in `post_states`, even untouched ones:

```rust
#[instruction]
pub fn update_score(
    player: AccountWithMetadata,        // modified
    config: AccountWithMetadata,        // read-only, NOT modified
    leaderboard: AccountWithMetadata,   // modified
    new_score: u64,
) -> LezResult {
    let player_data = PlayerAccount::try_from_slice(&player.data)?;
    let config_data = ConfigAccount::try_from_slice(&config.data)?;
    let leaderboard_data = LeaderboardAccount::try_from_slice(&leaderboard.data)?;

    // ... logic ...

    // ✅ Must include all three, even config which wasn't modified
    Ok(LezOutput::states_only(vec![
        player.into_post_state(updated_player_data.try_to_vec()?),
        config.into_post_state(config_data.try_to_vec()?),   // unchanged — still required
        leaderboard.into_post_state(updated_leaderboard.try_to_vec()?),
    ]))
}
```

> **⚠️ Warning:** If you use rest accounts (variadic), every account in that list must also be returned. This is a common source of off-by-one errors when the rest list length varies per call.

> **💡 Tip:** Write a helper wrapper that tracks whether an account was modified and always emits all accounts, using original data for unmodified ones. This makes it impossible to accidentally omit an account.

---

## 8. Accounts Come Before Args in Function Signatures

**Explanation:** The `#[instruction]` macro parses function signatures with a strict ordering requirement: all `AccountWithMetadata` parameters must come first, followed by all scalar/primitive arguments. Mixing the order causes a macro expansion error that often points at the wrong line.

**Symptoms:**
```
error: expected AccountWithMetadata, found u64
  --> src/lib.rs:42:5
   |
42 |     amount: u64,
   |     ^^^^^^^^^^^
```
The error points at the scalar arg, not the structural mistake.

**Fix:** Always put all accounts first, then all args:

```rust
// ❌ Broken: arg before account — breaks macro
#[instruction]
pub fn deposit(
    amount: u64,
    vault: AccountWithMetadata,
    depositor: AccountWithMetadata,
) -> LezResult { ... }

// ✅ Correct: accounts first, args after
#[instruction]
pub fn deposit(
    vault: AccountWithMetadata,
    depositor: AccountWithMetadata,
    amount: u64,
) -> LezResult { ... }
```

> **💡 Tip:** Adopt a consistent convention within each instruction: signer accounts first, then read-only accounts, then writable non-signer accounts, then scalar args. This is easy to read and avoids the ordering error.

---

## 9. Use `--cores 2 --max-jobs 2` to Avoid OOM During Proof Generation

**Explanation:** RISC Zero proof generation is extremely memory-intensive. By default, the prover uses all available CPU cores and launches multiple parallel proof jobs, easily exhausting RAM on a typical developer machine. When memory runs out, the OS OOM killer silently terminates the prover process and the CLI appears to hang indefinitely.

**Symptoms:**
- `spel` or `make proof` hangs for many minutes with no output
- System becomes unresponsive or starts swapping heavily
- No error message — the CLI stops making progress
- `dmesg | grep oom` shows OOM kill events

**Fix:** Limit parallelism during proof generation:

```bash
# Explicit flags
spel prove --cores 2 --max-jobs 2

# Or via environment variables (verify names for your spel version)
RISC0_PROVER_CORES=2 RISC0_MAX_JOBS=2 make proof
```

> **⚠️ Warning:** On a machine with 8 GB RAM, a single proof job can consume 4–6 GB. Two jobs may push you to the limit. If you still OOM, try `--cores 1 --max-jobs 1`.

> **💡 Tip:** For CI environments, set these flags globally in your CI config. Proof generation on runners with 4 GB RAM requires `--max-jobs 1`. Consider using a dedicated larger runner for proof generation jobs.

---

## 10. Rest Account Flag Is `--{name}` Not `--{name}-account`

**Explanation:** In `spel`, regular single accounts use `--{name}-account <addr>`. However, variadic "rest" accounts use a different flag format: `--{name}` (no `-account` suffix), and accept a comma-separated list of addresses.

**Symptoms:**
- `spel` errors with: `unexpected argument '--players-account'`
- Or the argument is silently accepted but only the first address is used
- Account count mismatch at the sequencer because rest accounts weren't passed correctly

**Fix:**

```bash
# ❌ Wrong: -account suffix for rest accounts
spel call --idl program.json --instruction update_scores \
  --players-account addr1,addr2,addr3

# ✅ Correct: no -account suffix, comma-separated list
spel call --idl program.json --instruction update_scores \
  --players addr1,addr2,addr3

# Regular single accounts still use -account:
spel call --idl program.json --instruction initialize \
  --authority-account <genesis-addr> \
  --config-account <pda-addr>
```

> **💡 Tip:** Check your IDL JSON to identify which parameters are rest accounts. They appear with a different type annotation (typically `rest` or `variadic`) compared to regular accounts.

---

## 11. `spel init` Scaffold Missing RISC Zero Metadata in `methods/Cargo.toml`

**Explanation:** The `spel init <name>` scaffold generates a workspace, but the generated `methods/Cargo.toml` omits the `[[package.metadata.risc0.methods]]` section that tells the RISC Zero build system which guest binaries to compile. Without this section, `cargo build` succeeds but does not produce the correct guest ELF binary.

**Symptoms:**
- `cargo build` succeeds with no errors
- `spel inspect` fails: cannot find guest binary
- The `target/riscv-guest/` directory is empty or missing expected ELF files
- Build log shows no RISC Zero guest compilation step

**Fix:** Add the metadata section to `methods/Cargo.toml` manually:

```toml
# methods/Cargo.toml

[package]
name = "methods"
version = "0.1.0"
edition = "2021"

[dependencies]
# ... your deps ...

# ✅ Add this section — it is missing from the scaffold
[[package.metadata.risc0.methods]]
name = "MY_PROGRAM"      # matches the constant name generated in build.rs
guest_path = "guest"     # path to the guest crate relative to methods/
```

> **⚠️ Warning:** The `name` field must match what your build scripts expect. Check how `make build` or `build.rs` references the guest binary name to confirm the correct value.

> **💡 Tip:** After fixing this, run `cargo clean` in the `methods/` directory and rebuild from scratch to avoid stale artifact confusion.

---

## 12. `sequencer_service` Takes a Config File Path, Not a Directory Path

**Explanation:** The `sequencer_service` binary expects a path to the JSON config **file** via `--config`, not a path to the directory containing it. Passing a directory path causes a startup error or silently uses default configuration.

**Symptoms:**
- `sequencer_service` exits immediately with a config parse error
- Or `sequencer_service` starts but uses unexpected default settings (wrong port, wrong RocksDB path)
- Config changes have no effect because the file is not being read

**Fix:**

```bash
# ❌ Wrong: path to the directory
./sequencer_service sequencer/service/configs/debug/

# ✅ Correct: path to the JSON config file
./sequencer_service --config sequencer/service/configs/debug/sequencer_config.json
```

> **💡 Tip:** Always point `--config` at the full file path, including the filename. The binary does not scan directories for config files.

---

## 13. Clean `target/riscv-guest/` After Dependency Changes

**Explanation:** RISC Zero maintains a separate build target directory for guest binaries (`target/riscv-guest/`). When you change dependencies in the guest crate, Cargo may not detect all stale artifacts in this directory, leading to linker errors or silently running an outdated binary.

**Symptoms:**
- Link errors after adding or removing guest dependencies: `undefined reference to ...`
- Program behaves as if an old version is running despite successful rebuild
- `make build` succeeds but `spel inspect` shows an outdated ImageID
- Errors referencing symbols that should no longer exist

**Fix:**

```bash
# Clear guest target artifacts before rebuilding after dependency changes
rm -rf target/riscv-guest/

# Then rebuild
make build
# or
cargo build --release
```

> **⚠️ Warning:** Never copy the `target/riscv-guest/` directory between projects. Each project's guest target is specific to its dependency tree and build configuration. Copying it causes subtle binary corruption that is very hard to diagnose.

> **💡 Tip:** Add `rm -rf target/riscv-guest/` to your `make clean` target so it always runs as part of a full clean. You can also add it as a pre-build step when you know dependencies changed.

---

## General Troubleshooting Reference

Use this table to quickly map symptoms to causes and fixes.

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `can't find crate for 'proc_macro'` | `borsh_derive` or other proc-macro in guest | Remove proc-macro crates from guest; use manual borsh impls |
| CLI hangs during proof generation | OOM on constrained host | Add `--cores 2 --max-jobs 2` |
| Account count mismatch at sequencer | Missing accounts in `post_states` | Return all received accounts, including unmodified ones |
| PDA address from CLI doesn't match sequencer | `spel pda` bug with integer seeds | Get address from sequencer logs on first `init` call |
| Commitment queries return empty results | `lssa` on broken revision `767b5af` | Switch to `main` branch |
| Mysterious commitment mismatch after restart | Stale RocksDB state | `just clean` from `~/lez` before restarting |
| Signature validation error with PDA signer | PDA passed where genesis account required | Use a genesis (keypair) account as signer |
| `init, mut` constraint conflict | Redundant constraint combination | Use only `#[account(init)]` |
| Macro error pointing at wrong line | Accounts and args out of order | Put all `AccountWithMetadata` params before scalar args |
| `--players-account` flag not recognized | Wrong flag format for rest accounts | Use `--players addr1,addr2` (no `-account` suffix) |
| Guest binary not produced by build | Missing risc0 metadata in `methods/Cargo.toml` | Add `[[package.metadata.risc0.methods]]` section |
| `sequencer_service` ignores config changes | Passing directory instead of file path | Pass the JSON file path to `--config`, e.g. `--config sequencer/service/configs/debug/sequencer_config.json` |
| Link errors after dependency change | Stale `target/riscv-guest/` | `rm -rf target/riscv-guest/` and rebuild |
| Account data zeroed after successful tx | `lssa` commitment RPC bug (`767b5af`) | Upgrade `lssa` to `main` branch |
| Signer check passes when it shouldn't | PDA silently bypasses signer constraint | Audit: ensure signer accounts are genesis accounts |
| `DeserializationError` reading account | Wrong account type or stale data | Check account type matches struct; reset state if stale |
| Build succeeds but no guest ELF produced | Missing risc0 metadata section | Add `[[package.metadata.risc0.methods]]` to `methods/Cargo.toml` |
| Old program behavior after code change | Stale `riscv-guest` artifacts | `rm -rf target/riscv-guest/` and rebuild |

> **💡 Tip:** When nothing on this table matches, enable verbose logging on `lssa` and run the failing operation again. The sequencer logs almost always contain the real error even when the client-side message is misleading.

> **⚠️ Warning:** Many LEZ errors produce well-formed responses with wrong content rather than explicit error codes. If a call "succeeds" but behaves incorrectly, add explicit logging of account contents immediately after deserialization — don't assume the account was read correctly.

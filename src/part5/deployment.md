# Deployment Checklist

This chapter covers local development workflow, testing strategies, security considerations, and a pre-deployment checklist for taking a LEZ program from "works on my machine" to a reproducible deployed state.

---

## Local Development Workflow

The standard local workflow:

```bash
# 1. Start sequencer (separate terminal)
sequencer_runner --port 3040

# 2. Build your program
make build

# 3. Generate IDL
make idl

# 4. Run local tests
cargo test

# 5. Deploy program
make setup
make deploy

# 6. Interact
make cli ARGS="initialize ..."
make cli ARGS="increment ..."
```

> **💡 Tip:** Keep the sequencer running in a dedicated terminal tab for the entire session. If you kill and restart it between steps, you'll need to re-deploy your program since the sequencer's state is cleared.

---

## Testing Patterns

### Unit Tests (Native)

Unit tests run on native (not riscv32im), so the full Rust testing toolkit is available:

```rust
// In your core crate
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_increment() {
        // Build mock accounts
        let counter = mock_account(CounterState::default());
        let authority = mock_signer_account();

        // Call initialize
        let output = counter::initialize(counter, authority).unwrap();

        // Verify
        let states = output.post_states();
        // ... assertions
    }
}
```

Because the core crate doesn't depend on the guest binary, you can run `cargo test` directly without firing up the zkVM. This makes unit tests fast — milliseconds, not seconds.

### Integration Tests with Live Sequencer

For end-to-end tests that exercise the full ZK proof pipeline:

```rust
#[tokio::test]
async fn test_counter_e2e() {
    let client = LezClient::new("http://localhost:3040");

    let counter_addr = client.create_account().await.unwrap();
    client.call_instruction(
        COUNTER_PROGRAM_ID,
        "initialize",
        vec![counter_addr, authority_addr],
        &[],
    ).await.unwrap();

    let state: CounterState = client.read_account(counter_addr).await.unwrap();
    assert_eq!(state.count, 0);
}
```

Integration tests require a running sequencer and a deployed program binary. Run them separately from unit tests:

```bash
# Unit tests (fast, no sequencer required)
cargo test --lib

# Integration tests (requires sequencer on :3040)
cargo test --test integration
```

### cfg-Gated Test Helpers

For mock account construction that only compiles in test mode:

```rust
#[cfg(test)]
pub fn mock_account<T: Default + BorshSerialize>(data: T) -> AccountWithMetadata<T> {
    AccountWithMetadata::new_for_test(
        [0u8; 32], // address
        data,
        1000,      // balance
        0,         // nonce
    )
}
```

---

## Security Considerations

### 1. Owner Checks

Always verify that the caller is authorized to modify an account. `#[account(signer)]` proves they control the key, but does not verify they're allowed to modify this specific state.

```rust
// ❌ Incomplete — proves key control but not authorization
#[account(signer)] caller: AccountWithMetadata<()>,
#[account(mut)] config: AccountWithMetadata<ConfigState>,

// ✅ Complete — check the stored admin address too
if &config.data().admin != caller.id() {
    return Err(LezError::InvalidSigner);
}
```

### 2. PDA Ownership

Use `#[account(owner = PROGRAM_ID)]` to ensure accounts you're modifying were created by your program. Without this, a caller could pass in an account from a different program with crafted data.

```rust
#[account(mut, owner = MY_PROGRAM_ID)] state: AccountWithMetadata<MyState>,
```

### 3. Input Validation

Validate numeric inputs — check for overflow and valid ranges. Rust's debug mode catches overflow panics, but release mode wraps silently.

```rust
// ❌ Silent overflow in release mode
let new_count = state.count + amount;

// ✅ Explicit overflow check
let new_count = state.count.checked_add(amount)
    .ok_or(LezError::Overflow)?;
```

### 4. Privacy Hygiene

When storing identifying information in public accounts, use commitment hashes rather than raw pubkeys to prevent correlation attacks:

```rust
// ❌ Raw pubkey — linkable on chain
pub owner: [u8; 32],

// ✅ Commitment — hides the owner identity
pub owner_commitment: [u8; 32], // SHA256(owner_pubkey || nonce)
```

### 5. Deterministic Operations

Everything in your program must be deterministic — the ZK proof requires it. Avoid wall-clock time, non-seeded randomness, and floating-point operations. Use BTreeMap instead of HashMap for ordered iteration.

---

## Pre-Deployment Checklist

Review each item before deploying to any shared environment:

- [ ] All instructions tested locally with the sequencer running
- [ ] IDL regenerated after the last code change (`make idl`)
- [ ] Version pinned: note which lssa/spel commits your build used
- [ ] Memory usage verified: built with `--jobs 2` on target hardware
- [ ] Account ownership verified: `#[account(owner = ...)]` on all accounts you mutate
- [ ] All accounts returned in `post_states` verified (no silent deletes)
- [ ] No `borsh_derive` imports in guest code (use `borsh` features = ["derive"])
- [ ] Signer accounts are genesis accounts (not PDAs)
- [ ] Client code regenerated from latest IDL
- [ ] Custom error codes documented in program README
- [ ] Arithmetic overflow checked with `checked_add`/`checked_mul` etc.
- [ ] No non-deterministic operations (no `SystemTime`, no HashMap iteration)

---

## Reproducible Builds

ImageID stability matters: if you deploy a program and the ImageID changes, existing clients break. Reproducible builds ensure byte-for-byte identical guest binaries across machines.

```bash
# Record exact dependencies
cargo generate-lockfile

# Build with locked dependencies
cargo build --release --locked --jobs 2

# Verify ImageID
cargo run --release -p idl-gen | jq '.program_id'
```

The ImageID will be the same as long as the guest binary is byte-for-byte identical. Using `--locked` ensures the same dependency versions across machines and CI runs.

> **💡 Tip:** Commit your `Cargo.lock` file to version control, even for libraries. For LEZ programs, a changing lock file means a changing ImageID means breaking deployed clients.

---

## Without Docker

The recommended path for containers and CI:

```bash
# Install rzup (RISC Zero's toolchain manager)
curl -L https://risczero.com/install | bash
rzup install

# Build with local toolchain (no Docker)
RISC0_DEV_MODE=0 cargo build --release --jobs 2
```

Do NOT use Docker-in-Docker for RISC Zero builds. The rzup local toolchain is reliable and works in standard Linux containers. Docker-in-Docker introduces permission and cgroup issues that are hard to debug and provide no benefit.

> **⚠️ Warning:** `RISC0_DEV_MODE=1` skips proof generation and uses fake proofs. This is useful for rapid local iteration but must NEVER be used for deployed programs. Always build with `RISC0_DEV_MODE=0` before deployment.

---

## Makefile Reference

A typical LEZ program Makefile has these targets:

```makefile
.PHONY: build idl setup deploy cli clean

build:
	CARGO_BUILD_JOBS=2 cargo build --release

idl:
	cargo run --release -p idl-gen > program.json

setup:
	# Create any required genesis accounts
	wallet account new

deploy:
	lez-cli deploy \
	  --program-binary target/riscv-guest/riscv32im-risc0-zkvm-elf/release/guest \
	  --idl program.json

cli:
	lez-cli call \
	  --idl program.json \
	  --instruction $(INSTRUCTION) \
	  $(ARGS)

clean:
	cargo clean
	rm -f program.json
```

Adjust paths for your workspace layout. The guest binary path comes from RISC Zero's build system — check your `methods/build.rs` output for the exact path.

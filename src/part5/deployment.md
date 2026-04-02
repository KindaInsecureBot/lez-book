# Deployment Checklist

This chapter covers the full workflow from a working local program to a deployed, verifiable LEZ program: build commands, setup, deploy, post-deploy verification, and what to know about production environments.

---

## Pre-Deployment Checklist

Work through this list before running `make deploy`. Each item has caused real deployment failures.

- [ ] **Program compiles to `riscv32im` without errors**
  Run `make build` or `cargo build --release` targeting `riscv32im-risc0-zkvm-elf`. Verify the guest ELF is produced in `target/riscv-guest/`. A successful host-side `cargo build` does not guarantee the guest compiles.

- [ ] **All instruction signatures have accounts before args**
  Every `#[instruction]` function must list all `AccountWithMetadata` parameters before any scalar arguments. Mixing order causes a macro error that points at the wrong line. Review each instruction signature.

- [ ] **All accounts returned in `post_states` (including unmodified ones)**
  The sequencer requires an exact account count match. For every instruction, count the input accounts and verify that `LezOutput::states_only(...)` returns exactly that many. Include read-only accounts with their original data.

- [ ] **`init` constraints do not also have `mut`**
  `#[account(init)]` is already mutable. Combining `init` and `mut` causes constraint conflicts. Search your code for `init, mut` or `mut, init` and remove the `mut`.

- [ ] **No `borsh_derive` proc-macros in guest code**
  `borsh_derive` does not compile for `riscv32im`. Audit your guest crate's `Cargo.toml` and all dependencies for proc-macro crates. All borsh serialization in guest code must be implemented manually.

- [ ] **PDA seeds verified against actual sequencer-assigned addresses**
  If your program uses integer seeds (u64, u128), `spel pda` may produce wrong addresses. Verify PDA addresses against sequencer logs from a test `init` call before hardcoding them anywhere.

- [ ] **Error codes defined (starting at 6000+)**
  Custom error codes must start at 6000 to avoid collision with framework error codes (1000–1999). Verify all `LezError::Custom(...)` values in your program use codes >= 6000.

- [ ] **IDL generated and reviewed**
  Run `make idl` and open the generated JSON. Verify instruction names, account names, argument types, and account ordering match your program. The IDL is what clients and `spel` use to call your program — a wrong IDL means broken clients.

- [ ] **`methods/Cargo.toml` has `[[package.metadata.risc0.methods]]` section**
  The `spel init` scaffold omits this section. Without it, the build may succeed without producing the correct guest binary. Confirm the section exists and the `name` and `guest_path` fields are correct.

- [ ] **`target/riscv-guest/` cleaned after any dependency changes**
  Stale artifacts cause hard-to-diagnose link errors. If you changed any dependencies since the last clean build, run `rm -rf target/riscv-guest/` before building for deployment.

---

## Build Workflow

### Docker Build (Recommended for Reproducibility)

The Docker build produces a reproducible guest binary — the same source always produces the same ImageID regardless of host toolchain version.

```bash
make build
```

This runs the build inside a pinned RISC Zero Docker image. The output is a deterministic guest ELF in `target/riscv-guest/`.

### Local Build (Faster for Development)

```bash
# Requires rzup and the correct RISC Zero toolchain installed
cargo build --release
```

Local builds are faster but the ImageID may differ from a Docker build if toolchain versions differ. Use Docker builds for any deployment you care about reproducibility on.

### Generate IDL

```bash
make idl
# Output: program.json (or as configured in your Makefile)
```

The IDL is a JSON schema describing all instructions, accounts, and argument types. Clients use this to call your program. Always regenerate the IDL after changing instruction signatures.

### Inspect ImageID

```bash
make inspect
# or
spel inspect --program ./target/riscv-guest/my_program.elf
```

This outputs the ImageID in three formats:
- **Hex**: raw bytes as a hex string
- **Decimal**: big-endian integer representation
- **ImageID**: the canonical identifier used as your program ID

Note the ImageID before deploying. If you deploy again after code changes, the ImageID will change — see [ImageID Change Implications](#imageid-change-implications) below.

---

## Setup and Deploy

### Setup: Create the Program Account

```bash
make setup
```

This creates a program account on `lssa`. Run this once for a new program. Running it again for an existing program is safe (it will error if the account already exists, which you can ignore).

Under the hood, this calls something like:

```bash
spel setup --program-id <image-id> --authority-account <genesis-addr>
```

### Deploy: Register the ImageID

```bash
make deploy
```

This registers your program's ImageID with the sequencer, making it callable. After this step, the sequencer will accept and verify transactions that invoke your program.

If you need to deploy manually (e.g., with a different key):

```bash
spel deploy \
  --program ./target/riscv-guest/my_program.elf \
  --program-id <image-id> \
  --authority-account <genesis-addr>
```

---

## Post-Deploy Verification

After deploying, verify the program is registered and callable before announcing it to users.

### Check Program Is Registered

```bash
spel inspect --program-id <image-id>
```

A successful response confirms the sequencer knows about your program. If this fails, the deploy did not complete successfully.

### Call Initialize

Most programs have an `initialize` instruction that sets up global state. Call it immediately after deploy:

```bash
make cli ARGS="initialize --authority-account <genesis-addr>"
# or directly:
spel call \
  --idl program.json \
  --instruction initialize \
  --authority-account <genesis-addr>
```

### Verify State Was Written

After initialization, inspect the PDA that `initialize` should have created:

```bash
wallet account inspect <pda-addr>
```

You should see non-empty account data. If the account is empty or not found, check:
1. The correct PDA address (see gotcha #3 about integer seeds)
2. That `lssa` is on a working revision (see gotcha #5 about `767b5af`)
3. That RocksDB state is not stale (see gotcha #4)

### Smoke Test Key Instructions

Call each major instruction at least once with known-good inputs before calling the deployment complete:

```bash
# Example for a token program
make cli ARGS="mint --authority-account <auth-addr> --vault-account <vault-addr> --amount 1000000"
make cli ARGS="transfer --from-account <from-addr> --to-account <to-addr> --amount 100"
```

---

## ImageID Change Implications

Every change to your program's guest binary — even a one-character comment change — produces a different ImageID. This is by design: the ImageID is a cryptographic commitment to the exact program code.

This means:

- **A redeployed program is a different program.** The new ImageID is a new program ID. Clients pointing at the old ImageID will continue calling the old program (if it still exists on the sequencer) or fail (if it was removed).

- **Accounts owned by the old ImageID cannot be written by the new one.** Account ownership is checked against the program ID at instruction execution time. If you deploy a new version, it cannot modify accounts created under the old version without an explicit migration.

**Options when upgrading a program:**

1. **Migration instruction:** Include a `migrate` instruction in the new program that accepts accounts owned by the old program ID and re-creates them under the new one. This requires careful access control.

2. **New program with data migration:** Deploy the new program, write a one-time migration script that reads old accounts (using the old program's IDL) and creates corresponding new accounts (using the new program), then shut down the old program.

3. **Accept the breaking change:** For programs in early development or with small user bases, a clean break may be simpler. Deploy fresh, reset state, update all client references to the new program ID.

> **🔄 Coming from Solidity?** In Solidity, you typically use proxy patterns (EIP-1967, UUPS) to upgrade contract logic while preserving state at a stable address. LEZ has no proxy pattern — the ImageID is the address and it's immutable. Design for upgrades from the start: keep state migration hooks in mind, or use a stable PDA owned by a governance program that can authorize migrations.

---

## Production Checklist

For deployments beyond local devnet, work through these additional items.

### Sequencer Configuration

- [ ] **Confirm sequencer liveness SLA.** The sequencer is the single point of liveness for your program. Understand the uptime guarantees for your lssa instance.
- [ ] **Verify sequencer is on a known-good revision.** Do not run production on `767b5af` or other known-broken commits. Pin to a tested revision and track it in your deployment docs.
- [ ] **Configure the health endpoint.** `lssa` exposes a health check endpoint. Set up monitoring to alert on downtime.
- [ ] **Review sequencer config for production settings.** Check RPC rate limits, max transaction size, and proof verification timeout values are appropriate for your expected load.

### Key Management

- [ ] **Back up your `nsk` (Nullifier Spending Key).** The `nsk` is the root secret for all private accounts. Loss of the `nsk` means permanent loss of access to private account contents. Store it in a hardware security module or offline cold storage.
- [ ] **Separate deployment keys from operational keys.** Use a dedicated genesis account for deployment that is separate from operational authority accounts. Restrict access to the deployment key.
- [ ] **Document all genesis account IDs** and their purposes. Losing the keypair for a genesis account means losing the ability to sign as that account permanently.

### RocksDB Backup Strategy

- [ ] **Set up regular RocksDB snapshots.** RocksDB supports online backups via the backup engine. Schedule these and store them off-host.
- [ ] **Test recovery from backup.** A backup you haven't tested restoring is not a backup. Verify the recovery procedure works before going to production.
- [ ] **Configure backup retention.** Keep enough historical snapshots to recover from both immediate failures and slow corruptions.

### Version Pinning

- [ ] **Pin `lssa` revision** in your deployment scripts and documentation.
- [ ] **Pin `spel` revision** used for deployment.
- [ ] **Record the exact build command** used to produce the deployed guest binary, including Docker image tag or `rzup` toolchain version.
- [ ] **Store the expected ImageID** alongside your pinned versions. Verify the ImageID matches during any re-build or re-deploy.

### Monitoring

- [ ] **Sequencer health endpoint:** Set up alerting if the health check fails.
- [ ] **Proof verification rate:** Monitor how many transactions are being accepted vs. rejected. A high rejection rate may indicate a client bug or a sequencer issue.
- [ ] **Commitment growth rate:** Track the number of new commitments per time window. Anomalies may indicate bugs or attacks.
- [ ] **RocksDB disk usage:** Set up disk space alerts. A full disk will crash `lssa`.

---

## Rollback

Because the ImageID is immutable and tied to the exact program binary, traditional "rollback" (reverting to a previous code version at the same address) is not directly possible. Instead:

**Rollback means redirecting clients to the previous program ID.**

1. Keep the old program's deployment artifacts (ELF binary, IDL, ImageID).
2. If the old program is still registered on the sequencer, clients can switch back immediately by pointing at the old program ID.
3. If the old program was deregistered, re-deploy it from the old artifacts. The ImageID will be identical to the original, so existing accounts owned by it are still valid.

```bash
# Re-deploy old version from archived artifacts
spel deploy \
  --program ./archive/v1.0/my_program.elf \
  --program-id <old-image-id> \
  --authority-account <genesis-addr>
```

> **💡 Tip:** Never delete old program binaries or IDLs. Archive every deployed version with its ImageID, IDL, and `lssa` revision. This is your rollback kit.

> **⚠️ Warning:** If the new program version wrote any state (new accounts, modified existing accounts), rollback will leave that state in place. The old program cannot read or write accounts it didn't create. Plan for state compatibility if there's any chance you'll need to roll back after users have interacted with the new version.

# Appendix A: CLI Reference

This appendix documents the command-line tools available for LEZ development: `lez-cli`, `wallet`, and `lez-client-gen`. For type formats when passing arguments, see [Appendix B: Type Format Table](./type-formats.md).

---

## lez-cli

The primary developer CLI for building, inspecting, deploying, and calling LEZ programs.

### Global Options

These options apply to most `lez-cli` subcommands:

| Flag | Short | Description |
|------|-------|-------------|
| `--idl <file>` | `-i` | Path to the IDL JSON file for the program |
| `--program <binary>` | `-p` | Path to the compiled guest ELF binary |
| `--program-id <id>` | | Program ID (ImageID) in hex, decimal, or ImageID format |
| `--dry-run` | | Simulate the call without submitting to the sequencer |

---

### `lez-cli init <name>`

Scaffold a new LEZ program workspace.

```bash
lez-cli init my_program
```

Creates a directory `my_program/` with the standard workspace structure:
```
my_program/
├── Cargo.toml          # workspace manifest
├── Makefile            # build/deploy shortcuts
├── methods/
│   ├── Cargo.toml      # ⚠️ Missing risc0 metadata section — add manually
│   └── guest/
│       └── src/
│           └── main.rs
└── program/
    └── src/
        └── lib.rs
```

> **⚠️ Warning:** The scaffold's `methods/Cargo.toml` is missing the `[[package.metadata.risc0.methods]]` section. Add it before building or the guest binary will not be produced. See [Gotchas #11](../part5/gotchas.md#11-lez-cli-init-scaffold-missing-risc-zero-metadata-in-methodscargotoml).

---

### `lez-cli build`

Compile the program to `riscv32im-risc0-zkvm-elf`.

```bash
lez-cli build
# equivalent to:
make build
```

Produces the guest ELF in `target/riscv-guest/`. Use `make build` for Docker-based reproducible builds. Use `lez-cli build` for local builds with the `rzup` toolchain.

---

### `lez-cli idl`

Generate the IDL (Interface Definition Language) JSON from the compiled program.

```bash
lez-cli idl
lez-cli idl --out program.json
lez-cli idl --out ./clients/program.json
```

| Flag | Description |
|------|-------------|
| `--out <file>` | Output file path (default: `program.json` in current directory) |

The IDL JSON describes all instructions, their account parameters, and argument types. Clients and `lez-cli` itself use this file to call the program correctly. Regenerate the IDL after any change to instruction signatures.

---

### `lez-cli inspect`

Show the ImageID (program ID) of the compiled guest binary.

```bash
lez-cli inspect
lez-cli inspect --program ./target/riscv-guest/my_program.elf
lez-cli inspect --program-id <id>
```

Output includes three formats:
- **Hex**: raw 32-byte hash as a hex string
- **Decimal**: big-endian integer
- **ImageID**: canonical identifier used as the program ID on the sequencer

Use `make inspect` as a shorthand if your Makefile is configured.

> **💡 Tip:** The ImageID changes with every code change, including comments and formatting. Always run `lez-cli inspect` after a build to confirm you have the ImageID you expect before deploying.

---

### `lez-cli pda`

Derive a PDA (Program Derived Address) from seeds and a program ID.

```bash
lez-cli pda --seeds '["my_seed", "another_seed"]' --program-id <id>
lez-cli pda --seeds '["state"]' --program-id <id>
```

Seeds are passed as a JSON array. Supported seed types:
- String seeds: `"my_seed"` — works correctly
- Bytes seeds: `[1, 2, 3]` — check your version for support

> **⚠️ Warning:** `lez-cli pda` produces incorrect addresses for `u64` and `u128` integer seeds. For programs with integer seeds, derive the address from sequencer logs during the first `init` call instead. See [Gotchas #3](../part5/gotchas.md#3-lez-cli-pda-gives-wrong-addresses-for-u64u128-seeds).

> **💡 Tip:** String seeds are always safe. Encode integer seeds as strings (`format!("user:{}", user_id)`) to avoid this bug.

---

### `lez-cli deploy`

Register the program's ImageID with the sequencer, making it callable.

```bash
lez-cli deploy
lez-cli deploy --program ./target/riscv-guest/my_program.elf
make deploy
```

Run this after `make setup` (which creates the program account). Deploying an already-deployed program is idempotent.

---

### Calling Instructions

Call a specific instruction on a deployed program.

```bash
lez-cli call \
  --idl program.json \
  --instruction <instruction_name> \
  --<account_name>-account <address> \
  [--<arg_name> <value> ...]
```

**Account flags:**
- Regular accounts: `--{name}-account <addr>` — one address per flag
- Rest (variadic) accounts: `--{name} <addr1,addr2,...>` — comma-separated, no `-account` suffix

**Argument flags:**
- `--<arg_name> <value>` for each scalar argument
- See [Appendix B](./type-formats.md) for how to format each type

**Examples:**

```bash
# Initialize with a single signer account
lez-cli call \
  --idl program.json \
  --instruction initialize \
  --authority-account 5Kg8cfY8iAiFEBzQPiTVPDLDTvBpMfQ6VFyiKNfJNQ3

# Deposit with two accounts and a u64 argument
lez-cli call \
  --idl program.json \
  --instruction deposit \
  --vault-account <vault-addr> \
  --depositor-account <depositor-addr> \
  --amount 1000000

# Batch update with rest accounts (variadic)
lez-cli call \
  --idl program.json \
  --instruction batch_update \
  --authority-account <auth-addr> \
  --targets addr1,addr2,addr3 \
  --value 42
```

**Makefile shorthand:**

Most projects configure a `make cli` target:

```bash
make cli ARGS="initialize --authority-account <genesis-addr>"
make cli ARGS="deposit --vault-account <vault> --depositor-account <dep> --amount 1000000"
```

---

### `--bin-<NAME>` Flag (CPI)

When calling an instruction that performs CPI (Cross-Program Invocation), pass the dependency program binary so `lez-cli` can include it in the call context:

```bash
lez-cli call \
  --idl program.json \
  --instruction cpi_instruction \
  --bin-token_program ./deps/token_program.elf \
  --authority-account <addr>
```

The `NAME` must match the program name as declared in the calling program's CPI invocation.

---

## wallet

The `wallet` CLI manages genesis accounts, private account scanning, and balance transfers. The wallet communicates with `lssa` via RPC.

### Environment Variable

```bash
export NSSA_WALLET_HOME_DIR=/path/to/wallet/config
```

Set this to a persistent directory. All wallet state (keys, account metadata) is stored here. If you lose this directory without a backup, you lose access to the private keys stored in it.

---

### `wallet init`

Initialize the wallet configuration directory.

```bash
wallet init
NSSA_WALLET_HOME_DIR=/my/wallet wallet init
```

Creates the config directory and generates the initial key material. Run this once per wallet instance.

---

### `wallet account new`

Create a new genesis account.

```bash
wallet account new
wallet account new private    # create a private (shielded) account
```

Genesis accounts are keypair-backed. The `private` variant creates a shielded account with a commitment-based identity. Both types return an account ID after creation.

> **💡 Tip:** The `nsk` (Nullifier Spending Key) is the root secret for all private accounts. Back it up immediately. See [Deployment: Key Management](../part5/deployment.md#key-management).

---

### `wallet account list`

List all accounts managed by this wallet.

```bash
wallet account list
```

Output includes account IDs, types (genesis/private), and balances if applicable.

---

### `wallet account inspect <id>`

Show detailed information about a specific account.

```bash
wallet account inspect 5Kg8cfY8iAiFEBzQPiTVPDLDTvBpMfQ6VFyiKNfJNQ3
wallet account inspect <pda-addr>
```

Displays account data, owner program ID, and balance. Useful for verifying state after calling an instruction. If this returns empty data for an account that should have been initialized, see [Gotchas #4](../part5/gotchas.md#4-rocksdb-stale-state-persists-across-lssa-restarts) and [Gotchas #5](../part5/gotchas.md#5-lssa-revision-767b5af-has-broken-commitment-rpc).

---

### `wallet account sync-private`

Scan the commitment tree for private account updates addressed to this wallet.

```bash
wallet account sync-private
```

This scans new commitments using the wallet's viewing keys and decrypts any that belong to this wallet. Use this to find incoming private transfers or check private account balances.

> **💡 Tip:** Private account scanning uses view tags to skip 99.6% of commitments. Even so, scanning may take time on a sequencer with many commitments. Run this after any expected incoming transfer.

---

### `wallet auth-transfer init`

Initialize the transfer authentication module for a private account. Required before sending private transfers.

```bash
wallet auth-transfer init --account-id <private-account-id>
```

This must be run once per private account before that account can send outgoing transfers.

---

### `wallet auth-transfer send`

Send a private token transfer between accounts.

```bash
wallet auth-transfer send \
  --from <sender-account-id> \
  --to <recipient-account-id> \
  --amount <amount>
```

The transfer is posted to the sequencer as a shielded transaction. The recipient must run `wallet account sync-private` to detect and claim it.

---

### `wallet check-health`

Verify connectivity to the sequencer.

```bash
wallet check-health
```

Useful for diagnosing connection issues before attempting transactions.

---

## lez-client-gen

`lez-client-gen` generates typed client code from a LEZ program IDL. Use it to produce safe, ergonomic client bindings instead of calling `lez-cli` directly from application code.

### Basic Usage

```bash
lez-client-gen \
  --idl program.json \
  --out-dir ./generated/
```

### Options

| Flag | Description |
|------|-------------|
| `--idl <file>` | Path to the IDL JSON file (required) |
| `--out-dir <dir>` | Output directory for generated files (required) |
| `--emit-header` | Emit a C header file alongside the Rust bindings |
| `--no-ffi` | Skip generating FFI-compatible types (pure Rust output only) |
| `--prefix <str>` | Prefix all generated type names with `<str>` to avoid collisions |

### Output

`lez-client-gen` produces Rust source files in `--out-dir` containing:

- Typed instruction builder structs for each instruction
- Account argument types matching the IDL
- Serialization/deserialization code for all argument types
- (With `--emit-header`) A C header for use from non-Rust clients

### Example

```bash
# Generate Rust client bindings
lez-client-gen --idl program.json --out-dir ./crates/program-client/src/generated/

# Generate with C header for FFI consumers
lez-client-gen \
  --idl program.json \
  --out-dir ./clients/ffi/ \
  --emit-header \
  --prefix MyProgram
```

After generation, add the generated directory as a module in your Rust crate:

```rust
// src/lib.rs
mod generated;
pub use generated::*;
```

> **⚠️ Warning:** Regenerate client bindings every time the IDL changes. Stale bindings that don't match the deployed program will cause runtime deserialization errors.

> **💡 Tip:** Commit generated bindings to your repository and set up CI to verify they are up-to-date with the IDL. A `make codegen && git diff --exit-code` check catches IDL drift early.

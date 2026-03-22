# Appendix A: CLI Reference

This appendix documents the command-line tools available for LEZ development.

---

## lez-cli

The primary developer CLI for building, deploying, and interacting with LEZ programs.

```
lez-cli [OPTIONS] <SUBCOMMAND>
```

**Global Options:**

| Flag | Description | Default |
|---|---|---|
| `--rpc-url <URL>` | Sequencer RPC URL | `http://localhost:3040` |
| `--wallet <PATH>` | Wallet directory path | `~/.lssa/wallet` |

---

### lez-cli init

Scaffold a new LEZ program workspace.

```bash
lez-cli init <name>
```

Creates a `<name>/` directory with the following workspace layout:

```
<name>/
├── core/           # Shared types and instruction logic (native + guest)
├── methods/        # RISC Zero guest crate
│   ├── guest/      # Guest binary entry point
│   └── build.rs    # Embeds guest ELF at compile time
├── idl-gen/        # Binary to generate the IDL JSON
├── cli-wrapper/    # Optional typed CLI wrapper
└── Makefile        # build / idl / deploy / cli targets
```

> **⚠️ Warning:** The scaffold may be missing `risc0-zkvm` metadata in `methods/Cargo.toml`. If your build fails after `lez-cli init`, manually add:
> ```toml
> [package.metadata.risc0]
> methods = ["guest"]
> ```

---

### lez-cli call

Call an instruction on a deployed program.

```bash
lez-cli call \
  --idl <idl.json> \
  --instruction <instruction-name> \
  --<account-name> <address> \
  [--<arg-name> <value>] \
  [--bin-<program-name> <path>]
```

**Flags:**

| Flag | Description |
|---|---|
| `--idl <path>` | Path to the program's IDL JSON file |
| `--instruction <name>` | Name of the instruction to call |
| `--<account-name> <addr>` | Account address (one flag per account) |
| `--<arg-name> <value>` | Argument value (one flag per arg) |
| `--bin-<name> <path>` | Path to a program binary (for CPI programs) |

**Example:**

```bash
lez-cli call \
  --idl ./counter.json \
  --instruction increment \
  --counter 4vKn7pW8XzR2mQ9aBcDe1F \
  --authority 7hJkLmN3pQ5rSt6uVwXy8Z \
  --amount 5
```

> **💡 Tip:** Account and argument flag names come directly from your instruction's parameter names. Check your IDL JSON for the exact names if you're unsure.

---

### lez-cli deploy

Deploy a program binary to the sequencer. Returns the program's ImageID.

```bash
lez-cli deploy \
  --program-binary <path-to-elf> \
  --idl <path-to-idl.json>
```

The ImageID printed by this command is your program's on-chain identifier. Save it — you'll need it for PDA derivation, CPI calls, and client configuration.

---

### lez-cli account

Read and list account state.

```bash
lez-cli account get <address>    # read account state
lez-cli account list             # list known accounts
```

**account get** — fetches and displays the raw account state at the given address. Output is the borsh-decoded data if a matching IDL type is found, otherwise raw hex.

**account list** — lists all accounts the CLI knows about (from the wallet or config).

---

### lez-cli pda

Compute a PDA address from seeds.

```bash
lez-cli pda \
  --program <program-id> \
  --seeds <seed1> <seed2> ...
```

> **⚠️ Warning:** `lez-cli pda` gives WRONG addresses when seeds contain `u64` or `u128` typed arguments. The CLI's serialization does not match the on-chain SHA-256 derivation for numeric types. Literal string seeds and account address seeds work correctly. See the Gotchas chapter for a workaround.

---

## lez-client-gen

Reads an IDL file and generates typed client code for interacting with a LEZ program.

```
lez-client-gen [OPTIONS]
```

**Options:**

| Flag | Description | Default |
|---|---|---|
| `--idl <path>` | IDL JSON file to read | — |
| `--out-dir <path>` | Output directory for generated code | `./generated` |
| `--target <rust\|c>` | Code generation target | `rust` |

**Example:**

```bash
lez-client-gen \
  --idl ./counter.json \
  --out-dir ./client/src/generated \
  --target rust
```

The generated Rust code provides typed structs and async functions matching each instruction. Regenerate whenever the IDL changes.

> **⚠️ Warning:** Never edit generated files by hand. They will be overwritten the next time `lez-client-gen` runs. Put customization in wrapper code that imports the generated types.

---

## wallet

The wallet CLI manages genesis accounts and private transfers.

```
wallet [OPTIONS] <SUBCOMMAND>
```

---

### wallet init

Initialize a new wallet at the default location (`~/.lssa/wallet`).

```bash
wallet init
```

---

### wallet account

Manage genesis and private accounts.

```bash
wallet account new                   # create a new genesis account
wallet account new private           # create a new private account
wallet account list                  # list all known accounts
wallet account sync-private          # scan chain for incoming private txs
```

**account new** — generates a new keypair and registers it as a genesis account with the sequencer.

**account new private** — generates a new account with an associated nsk/npk for privacy.

**account list** — prints all known accounts with their addresses, balances, and types.

**account sync-private** — scans recent blocks for private commitments whose view tags match your vpk. Decrypts matching commitments using your nsk and adds them to the local wallet. Run this whenever you're expecting to have received private funds or messages.

---

### wallet auth-transfer

Send and receive private (authenticated) transfers.

```bash
wallet auth-transfer init \
  --program <program-id> \
  --account <genesis-addr>

wallet auth-transfer send \
  --to <recipient-npk> \
  --amount <value> \
  --program <program-id>
```

**auth-transfer init** — initializes a private program account, linking a genesis account to the privacy system.

**auth-transfer send** — sends a private transfer to a recipient identified by their npk. The sequencer sees only a commitment; the transfer amount and recipient identity are hidden.

---

## sequencer_runner

The local sequencer for development. Not typically invoked directly in production — check the lssa repository for production deployment docs.

```bash
sequencer_runner --port 3040
```

**Options:**

| Flag | Description | Default |
|---|---|---|
| `--port <port>` | RPC listen port | `3040` |
| `--db-path <path>` | RocksDB state directory | `~/.lssa/db` |

> **⚠️ Warning:** When upgrading lssa versions, delete the RocksDB directory before restarting (`rm -rf ~/.lssa/db`). Stale state from a previous schema causes confusing errors. See the Gotchas chapter.

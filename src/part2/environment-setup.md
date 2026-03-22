# Environment Setup

This chapter walks through a fully reproducible development environment for LEZ on Ubuntu/Debian Linux. Every command is shown in full. Follow the steps in order — each step builds on the previous one.

---

## Prerequisites Overview

You will install:

- **Rust** (stable + nightly toolchains)
- **RISC Zero toolchain** via `rzup`
- **lssa** (local sequencer + wallet CLI) — built from source
- **lez-cli** (from the `spel` monorepo) — built from source

---

## Step 1: Rust

Install the Rust toolchain manager (`rustup`) and both the stable and nightly toolchains.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env
rustup toolchain install nightly
rustup default stable
```

Verify:

```bash
rustc --version
cargo --version
```

---

## Step 2: RISC Zero Toolchain

RISC Zero provides `rzup`, an installer that sets up the RISC Zero toolchain including `cargo-risczero` and the `riscv32im` compilation target.

```bash
curl -L https://risczero.com/install | bash
# Follow the installer's instructions to add rzup to your PATH, then reload:
source $HOME/.bashrc
rzup install
```

`rzup install` fetches the RISC Zero guest toolchain and registers the `riscv32im` Rust target locally. After this step, `cargo risczero` should be available.

Verify:

```bash
cargo risczero --version
```

> **⚠️ Warning:** If you're in a container or CI environment, make sure to use the rzup local toolchain path (not Docker). Docker-in-Docker setups are unreliable. rzup's local build path is the correct approach for containers and constrained environments.

---

## Step 3: Clone and Build lssa (Sequencer + Wallet)

`lssa` is the local development node for LEZ. It contains two binaries you need:

- **`sequencer_runner`**: the local sequencer that validates ZK proofs and processes transactions
- **`wallet`**: the CLI wallet for managing accounts and signing transactions

```bash
git clone https://github.com/nssaorg/lssa.git
cd lssa
# Build both binaries — use --jobs 2 to avoid OOM on constrained hosts
cargo build --release -p sequencer_runner -p wallet --jobs 2
```

> **⚠️ Warning:** Use `--jobs 2` on machines with limited RAM. RISC Zero guest compilation is extremely memory-hungry. OOM kills during compilation will silently produce corrupted or incomplete binaries without a clear error message. If your build fails unexpectedly, reduce parallelism first.

**A note on version pinning:** `lez-cli` pins to a specific `lssa` revision internally. If you encounter RPC errors when calling the sequencer, check which `lssa` revision `lez-cli` expects. The `main` branch is generally safe. Known bad revision: `767b5af` has a broken commitment RPC — avoid checking out that specific commit.

---

## Step 4: Clone and Build lez-cli

`lez-cli` lives in the `spel` monorepo alongside the SPEL macro framework and the client code generator.

```bash
git clone https://github.com/nssaorg/spel.git
cd spel
cargo build --release -p lez-cli -p lez-client-gen --jobs 2
# Add the built binaries to your PATH for the current session
export PATH="$PATH:$(pwd)/target/release"
```

To make this permanent, add the export line to your `~/.bashrc` or `~/.zshrc` with the absolute path:

```bash
echo 'export PATH="$PATH:/path/to/spel/target/release"' >> ~/.bashrc
source ~/.bashrc
```

**What's in the `spel` monorepo:**

- `spel` — the SPEL procedural macro framework (`#[lez_program]`, `#[instruction]`, etc.)
- `lez-cli` — the primary developer CLI for scaffolding, deploying, and calling programs
- `lez-client-gen` — generates typed client code (TypeScript or Rust) from a program IDL

---

## Step 5: Starting the Local Sequencer

The sequencer must be running before you can deploy or call any programs. Run it in a dedicated terminal.

```bash
# From the lssa directory
./target/release/sequencer_runner --port 3040
```

The sequencer listens on port `3040` by default. Leave this terminal open for the duration of your development session.

> **💡 Tip:** On first run, the sequencer creates its RocksDB state database in the current directory (or a configured path). If you upgrade `lssa` to a new version, delete the old state directory before starting the new sequencer — stale state from an older version causes subtle, hard-to-debug failures that look like valid RPC responses with wrong data. When in doubt, wipe the state and start fresh.

---

## Step 6: Wallet Setup

With the sequencer running, initialize your local wallet and create a genesis account to use for development.

```bash
# Initialize the wallet (creates key storage)
./target/release/wallet init

# Create a new genesis account (funded at chain genesis — usable immediately)
./target/release/wallet account new

# List accounts and confirm the new account appears with a balance
./target/release/wallet account list
```

**Genesis accounts vs. derived accounts:**

Genesis accounts are created directly by the wallet and are funded at chain initialization — they can sign transactions. Derived accounts (PDAs, program-derived addresses) are owned by programs and cannot sign transactions by themselves. When LEZ instructions require a `#[account(signer)]`, it must be a genesis account.

---

## Step 7: Verify Everything

Run these checks to confirm the full stack is operational:

```bash
# Sequencer health check (should return HTTP 200)
curl http://localhost:3040/health

# lez-cli version
lez-cli --version

# Wallet account list (should show your genesis account)
wallet account list
```

If all three commands succeed without errors, your environment is ready.

---

## Environment Variables

Set these in your shell so that `lez-cli` and other tools can find the sequencer and wallet without extra flags:

```bash
export LEZ_RPC_URL="http://localhost:3040"
export LEZ_WALLET_PATH="$HOME/.lssa/wallet"
```

Add both lines to your `~/.bashrc` or `~/.zshrc` so they persist across sessions:

```bash
echo 'export LEZ_RPC_URL="http://localhost:3040"' >> ~/.bashrc
echo 'export LEZ_WALLET_PATH="$HOME/.lssa/wallet"' >> ~/.bashrc
source ~/.bashrc
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `error: toolchain 'riscv32im...' not found` | RISC Zero guest toolchain not installed | Run `rzup install` again |
| Build OOM / silent binary corruption | Too many parallel compile jobs | Add `--jobs 2` to `cargo build` |
| `RocksDB error` on sequencer start | Stale state from old `lssa` version | Delete the RocksDB state directory and restart |
| RPC errors from `lez-cli` | `lssa` version mismatch | Check which `lssa` revision `lez-cli` expects; avoid rev `767b5af` |
| `lez-cli: command not found` | Binary not on PATH | Add `spel/target/release` to your `PATH` |
| `wallet: connection refused` | Sequencer not running | Start `sequencer_runner --port 3040` in a separate terminal |

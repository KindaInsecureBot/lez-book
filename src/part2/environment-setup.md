# Environment Setup

This chapter walks through a fully reproducible development environment for LEZ on Ubuntu/Debian Linux. Every command is shown in full. Follow the steps in order — each step builds on the previous one.

---

## Prerequisites Overview

There are two paths for getting a local sequencer running:

- **Quick start (Docker)**: Docker + Docker Compose, Rust, RISC Zero, `lez-cli`. Gets a sequencer running in minutes without building `sequencer_service` from source. Recommended for most developers.
- **From source (Advanced)**: Everything above, plus building `sequencer_service` and `wallet` from source. For contributors or when you need to modify the sequencer or wallet.

Both paths converge at `lez-cli` setup and use the same wallet CLI and verification steps.

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

## Step 3: Running the Sequencer (Docker)

This is the recommended path. Docker Compose starts the full local stack — sequencer, bedrock node, indexer, and explorer — without requiring you to build `sequencer_service` from source.

```bash
git clone https://github.com/logos-blockchain/logos-execution-zone.git ~/lez
cd ~/lez
docker compose up
```

The sequencer listens on port **3040**. The explorer is available at **http://localhost:8080**.

### Wallet Setup (Docker path)

The `wallet` binary is a client-side CLI, so you still build it from source. Building just `wallet` is much faster than building `sequencer_service` — no ZK circuit artifacts are needed, and no `--features standalone` flag is required.

```bash
cd ~/lez
cargo build --release -p wallet --jobs 2
export NSSA_WALLET_HOME_DIR="$HOME/lez/wallet/configs/debug"
./target/release/wallet check-health
```

Add the env var to your shell profile so it persists:

```bash
echo 'export NSSA_WALLET_HOME_DIR="$HOME/lez/wallet/configs/debug"' >> ~/.bashrc
source ~/.bashrc
```

**First-run wallet setup:** On the very first run, the wallet will prompt:

```
Persistent storage not found, need to execute setup / Input password:
```

Choose a password and press Enter. The wallet creates `storage.json` in your config directory. This is a one-time setup — subsequent runs use the stored credentials without prompting.

### Stopping and Cleaning Up (Docker)

To stop the stack and remove all container state:

```bash
docker compose down -v
```

The `-v` flag removes the Docker volumes. Omit it if you want to preserve state across restarts.

---

## Step 4: Clone and Build lez-cli

`lez-cli` lives in the `spel` monorepo alongside the SPEL macro framework and the client code generator.

```bash
git clone https://github.com/logos-co/spel.git ~/spel
cd ~/spel
cargo build --release -p lez-cli --jobs 2
# Add the built binary to your PATH for the current session
export PATH="$PATH:$HOME/spel/target/release"
```

To make this permanent, add the export line to your `~/.bashrc` or `~/.zshrc`:

```bash
echo 'export PATH="$PATH:$HOME/spel/target/release"' >> ~/.bashrc
source ~/.bashrc
```

**What's in the `spel` monorepo:**

- `spel` — the SPEL procedural macro framework (`#[lez_program]`, `#[instruction]`, etc.)
- `lez-cli` — the primary developer CLI for scaffolding, deploying, and calling programs
- `lez-client-gen` — generates typed client code (TypeScript or Rust) from a program IDL

---

## Step 5: Verify Everything

Run these checks to confirm the full stack is operational:

```bash
# Sequencer health check (via wallet)
wallet check-health

# lez-cli version
lez-cli --version

# Wallet account list (should show pre-configured genesis accounts)
wallet account ls
```

If all three commands succeed without errors, your environment is ready.

---

## Alternative: Building from Source

If you prefer to build and run the sequencer locally without Docker, or need to modify the sequencer or wallet code, follow these steps instead of Step 3 above. The result is the same stack — you just compile everything yourself.

### Installing ZK Circuit Artifacts

The build requires pre-compiled ZK circuit artifacts from `logos-blockchain-circuits`. The build script checks for these at compile time and will panic if they're missing.

**Download and install the circuits for your platform:**

For **Linux x86_64**:
```bash
cd /tmp
curl -L -O https://github.com/logos-blockchain/logos-blockchain-circuits/releases/download/v0.4.2/logos-blockchain-circuits-v0.4.2-linux-x86_64.tar.gz
mkdir -p ~/.logos-blockchain-circuits
tar -xzf logos-blockchain-circuits-v0.4.2-linux-x86_64.tar.gz -C ~/.logos-blockchain-circuits --strip-components=1
```

For **macOS Apple Silicon (aarch64)**:
```bash
cd /tmp
curl -L -O https://github.com/logos-blockchain/logos-blockchain-circuits/releases/download/v0.4.2/logos-blockchain-circuits-v0.4.2-macos-aarch64.tar.gz
mkdir -p ~/.logos-blockchain-circuits
tar -xzf logos-blockchain-circuits-v0.4.2-macos-aarch64.tar.gz -C ~/.logos-blockchain-circuits --strip-components=1
```

Alternatively, set the `LOGOS_BLOCKCHAIN_CIRCUITS` environment variable to point to the circuits directory if you prefer a different location:
```bash
export LOGOS_BLOCKCHAIN_CIRCUITS=/path/to/logos-blockchain-circuits
```

> **⚠️ Warning:** Without these circuit artifacts, `cargo build` will fail with a panic from the `logos-blockchain-pol` build script. The `--features standalone` flag does not bypass this requirement — the build script runs unconditionally.

Check available versions at: https://github.com/logos-blockchain/logos-blockchain-circuits/releases

### Clone and Build logos-execution-zone (Sequencer + Wallet)

The `logos-execution-zone` repo (cloned as `~/lez`) contains two binaries:

- **`sequencer_service`**: the local sequencer that validates ZK proofs and processes transactions
- **`wallet`**: the CLI wallet for managing accounts and signing transactions

```bash
git clone https://github.com/logos-blockchain/logos-execution-zone.git ~/lez
cd ~/lez
# Build both binaries — use --jobs 2 to avoid OOM on constrained hosts
cargo build --release --features standalone -p sequencer_service -p wallet --jobs 2
```

> **⚠️ Warning:** Use `--jobs 2` on machines with limited RAM. RISC Zero guest compilation is extremely memory-hungry. OOM kills during compilation will silently produce corrupted or incomplete binaries without a clear error message. If your build fails unexpectedly, reduce parallelism first.

**A note on version pinning:** `lez-cli` pins to a specific `logos-execution-zone` revision internally. If you encounter RPC errors when calling the sequencer, check which revision `lez-cli` expects. The `main` branch is generally safe. Known bad revision: `767b5af` has a broken commitment RPC — avoid checking out that specific commit.

### Starting the Local Sequencer

The sequencer must be running before you can deploy or call any programs. Run it in a dedicated terminal.

The recommended approach is to use `just`, which sets all required environment variables automatically:

```bash
cd ~/lez
just run-sequencer
```

Alternatively, run the binary directly:

```bash
# From the lez directory
cd ~/lez
RUST_LOG=info RISC0_DEV_MODE=1 ./target/release/sequencer_service --config sequencer/service/configs/debug/sequencer_config.json &
```

`RISC0_DEV_MODE=1` enables dev mode, which uses fake ZK proofs for fast local iteration. Without it, the sequencer requires real proof verification — slow and unnecessary during development. `RUST_LOG=info` gives useful log output.

The sequencer starts in the background and listens on the port defined in the debug config. Leave it running for the duration of your development session.

> **⚠️ Warning:** The sequencer takes a config **FILE** path via `--config`. Do not pass a directory path — it won't work. Always use `--config sequencer/service/configs/debug/sequencer_config.json`, pointing at the JSON file itself.

> **💡 Tip:** On first run, the sequencer creates its RocksDB state database in the repo directory. If you upgrade to a new version or things get weird, run `just clean` (from `~/lez`) to remove stale state and start fresh. See the [Cleaning Up](#cleaning-up) section below.

### Wallet Setup (Source path)

The debug configuration in `logos-execution-zone` comes with **pre-configured genesis accounts** that are already funded. You just point the wallet at the debug config directory.

```bash
# Point the wallet at the debug configuration (pre-funded genesis accounts)
export NSSA_WALLET_HOME_DIR="$HOME/lez/wallet/configs/debug"

# Add to ~/.bashrc to persist across sessions
echo 'export NSSA_WALLET_HOME_DIR="$HOME/lez/wallet/configs/debug"' >> ~/.bashrc

# Verify the sequencer is reachable
wallet check-health

# List the pre-configured genesis accounts (should show funded accounts)
wallet account ls
```

**First-run wallet setup:** On the very first run, the wallet will prompt:

```
Persistent storage not found, need to execute setup / Input password:
```

Choose a password and press Enter. The wallet creates `storage.json` in your config directory. This is a one-time setup — subsequent runs use the stored credentials without prompting.

The recommended approach for wallet commands is via `just`, which automatically sets `NSSA_WALLET_HOME_DIR`:

```bash
just run-wallet check-health
just run-wallet account ls
```

**Genesis accounts vs. derived accounts:**

The debug config ships with pre-funded genesis accounts you can use immediately for development. Derived accounts (PDAs, program-derived addresses) are owned by programs and cannot sign transactions by themselves. When LEZ instructions require a `#[account(signer)]`, it must be a genesis account.

Once the wallet is set up, continue with **Step 4** (lez-cli) and **Step 5** (Verify Everything) above.

---

## Using `just` (Recommended)

The `~/lez` repo ships a `Justfile` with commands that handle environment setup automatically. These are the recommended way to run the sequencer and wallet when building from source:

```bash
# Run the sequencer (sets RISC0_DEV_MODE=1 and RUST_LOG=info automatically)
just run-sequencer

# Run wallet commands (sets NSSA_WALLET_HOME_DIR automatically)
just run-wallet check-health
just run-wallet account ls
```

The manual invocations shown earlier in this chapter are equivalent — `just` is a convenience wrapper that avoids having to re-export env vars each session.

---

## Cleaning Up

When upgrading `logos-execution-zone` to a new version, or if the sequencer behaves unexpectedly, run `just clean` to wipe all runtime state:

```bash
cd ~/lez
just clean
```

This removes:
- `sequencer/service/rocksdb` — sequencer state
- `sequencer/service/bedrock_signing_key` — signing key
- `indexer/service/rocksdb` — indexer state
- `wallet/configs/debug/storage.json` — wallet persistent storage

After `just clean`, the next `just run-sequencer` starts from a blank slate, and the next `just run-wallet` (or `wallet`) command prompts for a new password to recreate `storage.json`.

> **⚠️ Warning:** `just clean` deletes all local sequencer and wallet state. This is safe on a local dev setup. Never run it against an environment with state you want to keep.

For the Docker path, use `docker compose down -v` instead.

---

## Environment Variables

The one required environment variable is `NSSA_WALLET_HOME_DIR`, which points the wallet CLI at the correct configuration directory:

```bash
export NSSA_WALLET_HOME_DIR="$HOME/lez/wallet/configs/debug"
```

Add this to your `~/.bashrc` or `~/.zshrc` so it persists across sessions:

```bash
echo 'export NSSA_WALLET_HOME_DIR="$HOME/lez/wallet/configs/debug"' >> ~/.bashrc
source ~/.bashrc
```

> **⚠️ Warning:** There are no `LEZ_RPC_URL` or `LEZ_WALLET_PATH` environment variables. The wallet finds the sequencer via the config files in `NSSA_WALLET_HOME_DIR`. Do not set those fictional env vars — they have no effect.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `error: toolchain 'riscv32im...' not found` | RISC Zero guest toolchain not installed | Run `rzup install` again |
| Build OOM / silent binary corruption | Too many parallel compile jobs | Add `--jobs 2` to `cargo build` |
| `RocksDB error` on sequencer start | Stale state from old version | Run `just clean` from `~/lez` and restart |
| RPC errors from `lez-cli` | `logos-execution-zone` version mismatch | Check which revision `lez-cli` expects; avoid rev `767b5af` |
| `lez-cli: command not found` | Binary not on PATH | Add `$HOME/spel/target/release` to your `PATH` |
| `wallet: connection refused` | Sequencer not running | Start sequencer via `docker compose up` (Docker) or `just run-sequencer` (source) |
| `wallet check-health` fails | `NSSA_WALLET_HOME_DIR` not set or wrong path | Set `export NSSA_WALLET_HOME_DIR="$HOME/lez/wallet/configs/debug"` |
| `wallet account ls` shows nothing | Wrong config directory | Verify `NSSA_WALLET_HOME_DIR` points to `lez/wallet/configs/debug` |
| `logos-blockchain-pol` build script panics | Missing ZK circuit artifacts | Install `logos-blockchain-circuits` to `~/.logos-blockchain-circuits` (source path only — not needed for Docker) |

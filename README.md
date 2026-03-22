# LEZ Development Guide

A comprehensive guide for EVM/Solidity developers transitioning to LEZ/SPEL on-chain program development.

## What is LEZ?

**LEZ** (Logos Execution Zone) is the smart contract execution layer of the **Logos Network** — a decentralized governance and infrastructure stack built by **IFT (Institute of Free Technology)**.

- **Website**: [logos.co](https://logos.co)
- **GitHub (framework)**: [github.com/logos-co](https://github.com/logos-co) — SPEL framework and lez-cli (`spel` repo)
- **GitHub (sequencer)**: [github.com/logos-blockchain](https://github.com/logos-blockchain) — `lssa` (sequencer + wallet)

Every LEZ program runs inside the RISC Zero zkVM and generates a cryptographic proof. Programs are written in Rust using the SPEL macro framework. The underlying sequencer architecture is called **NSSA** (Nondeterministic State Sequencer Architecture).

📖 **[Read the book →](https://kindainsecurebot.github.io/lez-book/)**

## What's Inside

### Part 1: Understanding LEZ
- What is LEZ and how it differs from EVM
- Complete concept mapping: Solidity → SPEL

### Part 2: Getting Started
- Full environment setup (Rust, RISC Zero, local sequencer)
- Step-by-step Counter program tutorial

### Part 3: Core Concepts
- Accounts & State (the biggest mental shift from EVM)
- PDAs (Program Derived Addresses)
- Instructions & Validation
- The IDL

### Part 4: Advanced Topics
- Cross-Program Invocation
- Privacy-Preserving Transactions
- Private State Attestations
- Client Code Generation

### Part 5: Practical Guide
- Patterns & Recipes
- Gotchas & Troubleshooting (battle-tested)
- Deployment Checklist

## Building Locally

```bash
# Install mdbook
cargo install mdbook

# Serve locally with hot reload
mdbook serve --open
```

## Related Projects

- [SPEL Framework](https://github.com/logos-co/spel) — The Rust framework for building LEZ programs
- [LSSA](https://github.com/logos-blockchain/lssa) — Sequencer and wallet for local development
- [CryptoLizards](https://kindainsecurebot.github.io/cryptolizards/) — Interactive LEZ tutorial (CryptoZombies style)
- [SPEL Agent Resources](https://github.com/KindaInsecureBot/spel-agent-resources) — Comprehensive SPEL documentation

## Contributing

This guide is community-maintained. If you find errors or want to add content, PRs are welcome.

## License

MIT

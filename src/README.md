# LEZ Development Guide

Welcome to the **LEZ Development Guide** — a comprehensive reference for EVM/Solidity developers transitioning to LEZ/SPEL smart contract development.

## Who This Is For

You've written Solidity. You understand EVM concepts: contracts, storage slots, `msg.sender`, mappings, events, the ABI. You want to build on LEZ — a privacy-native smart contract platform powered by RISC Zero zero-knowledge proofs.

This book is your translation layer.

## What You'll Learn

- How LEZ concepts map to (and differ from) EVM concepts
- How to set up a complete local development environment
- How to build, deploy, and interact with SPEL programs
- How accounts, PDAs, and state management work
- How LEZ's privacy system works — commitments, nullifiers, view tags
- How to write ZK attestations about private state
- Patterns, gotchas, and everything that will bite you

## The Pitch

LEZ's killer feature is **privacy-native smart contracts**. The same program binary handles both public and private transactions. Users can prove facts about their private state (e.g., "my balance is ≥ 1000 tokens") without revealing the actual value. This isn't a layer on top — it's the base design.

If you've looked at Tornado Cash or Aztec and wished the entire contract layer worked that way: this is it.

## How to Read This Book

**Linear**: If you're new to LEZ, read Parts 1-3 in order. They build on each other.

**Reference**: Parts 4-5 and Appendices are reference material. Jump to what you need.

**Skim the Mental Model chapter first** (Chapter 2) even if you plan to read linearly. It gives you the concept map that makes everything else click faster.

## Source

All code examples in this book are based on real, tested programs. The running examples are:

- **Counter** — the hello world: `initialize` + `increment`
- **balance-attestation** — ZK proof that private balance ≥ threshold
- **logos-land** — hex grid land registry with PDAs, adjacency enforcement, and complex attestations

Every command shown is a command that has actually been run and verified.

---

Let's get started.

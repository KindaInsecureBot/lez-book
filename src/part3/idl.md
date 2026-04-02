# The IDL

## What Is the IDL?

The IDL (Interface Definition Language file) is a JSON file that describes your program's instructions, account types, argument types, and return types. It's the LEZ equivalent of a Solidity ABI.

> **🔄 Coming from Solidity?** If you've used `abi.json` files to call Solidity contracts, the IDL serves the same purpose. It's the machine-readable interface that tools use to encode/decode calls to your program.

## What the IDL Contains

A complete IDL includes compatibility fields for lssa-lang tooling. Here's a realistic example for a Counter program:

```json
{
  "name": "counter",
  "version": "0.1.0",
  "program_id": "<risc-zero-image-id>",
  "instructions": [
    {
      "name": "initialize",
      "discriminator": [175, 175, 109, 31, 13, 152, 155, 237],
      "execution": "sequential",
      "variant": "standard",
      "accounts": [
        { "name": "counter", "writable": true, "init": true },
        { "name": "owner", "signer": true }
      ],
      "args": [],
      "visibility": [0, 0]
    },
    {
      "name": "increment",
      "discriminator": [11, 18, 104, 9, 104, 174, 59, 33],
      "execution": "sequential",
      "variant": "standard",
      "accounts": [
        { "name": "counter", "writable": true },
        { "name": "caller", "signer": true }
      ],
      "args": [],
      "visibility": [0, 0]
    }
  ],
  "types": [
    {
      "name": "CounterState",
      "type": {
        "kind": "struct",
        "fields": [
          { "name": "count", "type": "u64" },
          { "name": "owner", "type": { "array": ["u8", 32] } }
        ]
      }
    }
  ]
}
```

### lssa-lang Compatibility Fields

| Field | Value | Meaning |
|---|---|---|
| `discriminator` | `[u8; 8]` | First 8 bytes of `SHA256("global:<instruction_name>")` |
| `execution` | `"sequential"` | How the instruction is executed (currently always sequential) |
| `variant` | `"standard"` | Instruction type (standard vs. specialized) |
| `visibility` | `[u8]` | Per-account visibility mask (0=public, 1=private-auth, 2=private-unauth) |

**Discriminator computation:**

```
discriminator = SHA256("global:" || instruction_name)[0..8]
```

For `"initialize"`: `SHA256("global:initialize")` → first 8 bytes = `[175, 175, 109, 31, 13, 152, 155, 237]`

The discriminator is used by spel and client code to identify which instruction to call. It's the LEZ equivalent of Solidity's 4-byte function selector.

## How the IDL Is Generated

The `generate_idl!` macro in your `idl-gen` crate produces the IDL. The macro scans your crate for exactly one `#[lez_program]` module — if it finds zero or more than one, it errors.

```rust
// idl-gen/src/main.rs
fn main() {
    let idl = counter_core::generate_idl!();
    println!("{}", serde_json::to_string_pretty(&idl).unwrap());
}
```

Build it:

```bash
make idl
# or equivalently:
cargo run --release -p idl-gen > counter.json
```

> **⚠️ Warning:** The `generate_idl!` macro finds exactly one `#[lez_program]` module in the crate. If you have multiple programs in one crate (not recommended), this will fail. Keep one program per crate.

## Never Edit by Hand

> **⚠️ Warning:** Never manually edit the IDL file. It's generated from your Rust code. If the IDL and your Rust code get out of sync, spel will encode arguments incorrectly and you'll get mysterious failures. Always regenerate after changing your program.

## How spel Uses the IDL

When you run:

```bash
spel call --idl counter.json --instruction increment --counter <addr> --caller <addr>
```

spel reads the IDL to know:

- Which arguments the instruction expects
- How to borsh-encode them
- Which accounts are signers, which are writable
- The program ID to send the transaction to
- The discriminator to identify the instruction

## How spel-client-gen Uses the IDL

`spel-client-gen` reads the IDL to generate typed client code:

```bash
spel-client-gen --idl counter.json --out-dir ./client
```

This generates Rust client code with typed function calls for each instruction. See the Client Code Generation chapter for details.

## IDL Versioning

The IDL is tied to a specific compiled binary (ImageID). When you redeploy with changes:

1. Rebuild the binary (`cargo build --release`)
2. Regenerate the IDL (`make idl`)
3. Redeploy (`make deploy`)
4. Regenerate any client code

The new ImageID in the IDL will be different if the program logic changed.

> **💡 Tip:** Commit your `counter.json` IDL file to source control alongside your program. This makes it easy to track which IDL corresponds to which deployed version, and gives downstream clients a stable reference to the interface.

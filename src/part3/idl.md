# The IDL

## What Is the IDL?

The IDL (Interface Definition Language file) is a JSON file that describes your program's instructions, account types, argument types, and return types. It's the LEZ equivalent of a Solidity ABI.

> **🔄 Coming from Solidity?** If you've used `abi.json` files to call Solidity contracts, the IDL serves the same purpose. It's the machine-readable interface that tools use to encode/decode calls to your program.

## What the IDL Contains

```json
{
  "name": "counter",
  "version": "0.1.0",
  "program_id": "<risc-zero-image-id>",
  "instructions": [
    {
      "name": "initialize",
      "accounts": [
        { "name": "counter", "writable": true, "init": true },
        { "name": "owner", "signer": true }
      ],
      "args": []
    },
    {
      "name": "increment",
      "accounts": [
        { "name": "counter", "writable": true },
        { "name": "caller", "signer": true }
      ],
      "args": []
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

## How the IDL Is Generated

The `generate_idl!` macro in your `idl-gen` crate produces the IDL:

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
# or:
cargo run --release -p idl-gen > counter.json
```

## Never Edit by Hand

> **⚠️ Warning:** Never manually edit the IDL file. It's generated from your Rust code. If the IDL and your Rust code get out of sync, lez-cli will encode arguments incorrectly and you'll get mysterious failures. Always regenerate after changing your program.

## How lez-cli Uses the IDL

When you run:

```bash
lez-cli call --idl counter.json --instruction increment --counter <addr> --caller <addr>
```

lez-cli reads the IDL to know:

- Which arguments the instruction expects
- How to borsh-encode them
- Which accounts are signers, which are writable
- The program ID to send the transaction to

## How lez-client-gen Uses the IDL

`lez-client-gen` reads the IDL to generate typed client code:

```bash
lez-client-gen --idl counter.json --out-dir ./client
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

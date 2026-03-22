# Appendix B: Type Format Table

This appendix documents how to pass argument values to `lez-cli`'s `--<arg-name>` flags, and how types are serialized on the wire.

---

## CLI Argument Formats

When calling instructions with `lez-cli call`, scalar arguments are passed as strings on the command line and decoded by the CLI according to their IDL types.

| IDL Type | CLI Format | Example |
|---|---|---|
| `u8` | decimal integer | `255` |
| `u16` | decimal integer | `1000` |
| `u32` | decimal integer | `100000` |
| `u64` | decimal integer | `18446744073709551615` |
| `u128` | decimal integer | `340282366920938463463374607431768211455` |
| `i64` | decimal integer (signed) | `-42` |
| `bool` | `true` or `false` | `true` |
| `string` | quoted string | `"hello"` |
| `[u8; 32]` | hex string (64 hex chars) | `deadbeef0102...` |
| `Vec<u8>` | hex string (variable length) | `cafebabe` |
| `[u8; N]` | hex string (2N hex chars) | `0102030405...` |
| Account address | base58 or hex | `4vKn7pW8X...` |

### Notes

- Account addresses can be passed as hex or base58 depending on the wallet and CLI version in use. Check `wallet account list` to see which format your accounts are stored in.
- For `[u8; 32]` values used as pubkeys or hashes, use the 64-character hex representation from `wallet account list`.
- `u64` and `u128` PDA seeds have a known encoding discrepancy in `lez-cli pda` — see the Gotchas chapter for details and a workaround.
- Hex strings are case-insensitive (`DEADBEEF` and `deadbeef` are equivalent).
- Do not include a `0x` prefix for hex values unless the CLI version you're using explicitly requires it.

---

## On-Wire Borsh Encoding

LEZ uses [Borsh](https://borsh.io/) (Binary Object Representation Serializer for Hashing) for all state serialization. Borsh is deterministic, which is required for ZK proof generation.

### Primitive Types

| Rust Type | Borsh Encoding | Size |
|---|---|---|
| `u8` | little-endian byte | 1 byte |
| `u16` | little-endian | 2 bytes |
| `u32` | little-endian | 4 bytes |
| `u64` | little-endian | 8 bytes |
| `u128` | little-endian | 16 bytes |
| `i8` | little-endian (signed) | 1 byte |
| `i16` | little-endian (signed) | 2 bytes |
| `i32` | little-endian (signed) | 4 bytes |
| `i64` | little-endian (signed) | 8 bytes |
| `i128` | little-endian (signed) | 16 bytes |
| `bool` | `0x00` or `0x01` | 1 byte |
| `f32` | little-endian IEEE 754 | 4 bytes |
| `f64` | little-endian IEEE 754 | 8 bytes |

### Composite Types

| Rust Type | Borsh Encoding |
|---|---|
| `[T; N]` | N consecutive borsh-encoded `T` values, no length prefix |
| `Vec<T>` | `u32` length prefix (little-endian) followed by N borsh-encoded `T` values |
| `String` | `u32` byte-length prefix followed by UTF-8 bytes |
| `Option<T>` | `0x00` for None; `0x01` followed by borsh-encoded `T` for Some |
| `enum` | `u8` variant index (0-based) followed by any variant fields |
| struct | Fields encoded in declaration order, no padding, no separators |

### Example: Encoding a Struct

```rust
#[derive(BorshSerialize, BorshDeserialize)]
pub struct TileState {
    pub owner_commitment: [u8; 32],   // 32 bytes
    pub claimed_at: u64,              //  8 bytes
    pub terrain: u8,                  //  1 byte
    pub elevation: u8,                //  1 byte
}
// Total: 42 bytes, no padding
```

Wire encoding for a `TileState` with all zeros except `terrain = 3`:

```
00000000 00000000 00000000 00000000   # owner_commitment bytes 0-15
00000000 00000000 00000000 00000000   # owner_commitment bytes 16-31
0000000000000000                      # claimed_at = 0 (u64 LE)
03                                    # terrain = 3
00                                    # elevation = 0
```

---

## PDA Seed Encoding

PDA addresses are computed as `SHA256(program_id || seed1_bytes || seed2_bytes || ...)` where each seed is encoded as follows:

| Seed Type (SPEL macro) | Encoding |
|---|---|
| `literal("foo")` | UTF-8 bytes of the string (no length prefix) |
| `account("name")` | raw 32-byte account address |
| `arg("name")` where arg is `u64` | 8 bytes little-endian |
| `arg("name")` where arg is `u128` | 16 bytes little-endian |
| `arg("name")` where arg is `[u8; 32]` | raw 32 bytes |

> **⚠️ Warning:** `lez-cli pda` does not correctly encode `u64` and `u128` typed seeds. Manual PDA computation using SHA256 with the encoding above will give correct results. Literal string seeds and account address seeds work correctly in the CLI.

---

## IDL Type Names

IDL JSON files use these type strings to describe instruction arguments and account data fields:

| IDL JSON `type` | Rust Type |
|---|---|
| `"u8"` | `u8` |
| `"u16"` | `u16` |
| `"u32"` | `u32` |
| `"u64"` | `u64` |
| `"u128"` | `u128` |
| `"i64"` | `i64` |
| `"bool"` | `bool` |
| `"string"` | `String` |
| `{"array": ["u8", 32]}` | `[u8; 32]` |
| `{"vec": "u8"}` | `Vec<u8>` |
| `{"option": "u64"}` | `Option<u64>` |
| `{"defined": "MyStruct"}` | `MyStruct` (defined in the IDL types section) |

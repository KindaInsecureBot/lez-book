# Appendix B: Type Format Table

This appendix documents how to pass argument values to `lez-cli`'s `--<arg-name>` flags. The format depends on the LEZ type as declared in your program's IDL.

---

## Scalar Types

| LEZ Type | CLI Format | Notes | Example |
|----------|-----------|-------|---------|
| `u8` | Decimal integer | 0–255 | `255` |
| `u16` | Decimal integer | 0–65535 | `1000` |
| `u32` | Decimal integer | 0–4294967295 | `1000000` |
| `u64` | Decimal integer | 0–18446744073709551615 | `18446744073709551615` |
| `u128` | Decimal integer | Very large values | `340282366920938463463374607431768211455` |
| `i8` | Decimal integer (may be negative) | -128–127 | `-42` |
| `i32` | Decimal integer (may be negative) | | `-1000` |
| `i64` | Decimal integer (may be negative) | | `-999999` |
| `bool` | `true` or `false` (lowercase) | Not 0/1 | `true` |
| `String` | Bare string (no quotes on CLI) | Spaces require shell quoting | `hello` |

---

## Byte Array Types

| LEZ Type | CLI Format | Notes | Example |
|----------|-----------|-------|---------|
| `[u8; 32]` | Lowercase hex string, no `0x` prefix | Exactly 64 hex characters | `deadbeef...` (64 chars) |
| `[u8; N]` | Lowercase hex string, no `0x` prefix | Exactly N*2 hex characters | 16 bytes = 32 hex chars |
| `Vec<u8>` | Comma-separated decimal bytes | Each value 0–255 | `1,2,3,4` |

**Examples for `[u8; 32]` (a 32-byte hash or public key):**

```bash
# Correct: 64 lowercase hex chars, no 0x prefix
--hash deadbeef0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c

# Wrong: has 0x prefix
--hash 0xdeadbeef...

# Wrong: uppercase hex
--hash DEADBEEF...
```

**Examples for `Vec<u8>`:**

```bash
# Four bytes: 1, 2, 3, 4
--data 1,2,3,4

# Empty byte vec
--data ""

# Single byte
--data 255
```

---

## Account Types

| LEZ Type | CLI Format | Notes | Example |
|----------|-----------|-------|---------|
| `AccountId` | Base58 encoded | Same format as Solana addresses | `5Kg8cfY8iAiFEBzQPiTVPDLDTvBpMfQ6VFyiKNfJNQ3` |

Accounts are passed using dedicated flags, not `--arg` style:

```bash
# Regular (non-variadic) account — use --{name}-account
--authority-account 5Kg8cfY8iAiFEBzQPiTVPDLDTvBpMfQ6VFyiKNfJNQ3
--vault-account      AbCdEfGh1234...

# Each account gets its own flag
lez-cli call \
  --idl program.json \
  --instruction transfer \
  --from-account <addr1> \
  --to-account <addr2> \
  --amount 1000
```

---

## Passing Multiple Accounts

### Regular Accounts (Non-Variadic)

Regular accounts use one flag per account: `--{name}-account <addr>`. You cannot pass multiple addresses to a single `--{name}-account` flag.

```bash
# ✅ Correct: one flag per account
lez-cli call --idl p.json --instruction my_ix \
  --player-account <addr1> \
  --config-account <addr2> \
  --vault-account <addr3>
```

### Variadic (Rest) Accounts

Rest accounts use `--{name}` (no `-account` suffix) with a comma-separated list:

```bash
# ✅ Correct: comma-separated, no spaces around commas, no -account suffix
lez-cli call --idl p.json --instruction batch_update \
  --authority-account <auth-addr> \
  --targets addr1,addr2,addr3,addr4

# ❌ Wrong: -account suffix on rest parameter
  --targets-account addr1,addr2

# ❌ Wrong: separate flags for each rest account
  --targets addr1 \
  --targets addr2
```

> **⚠️ Warning:** Mixing up the flag format for rest accounts is a common mistake. Check your IDL JSON to confirm which parameters are rest accounts. See [Gotchas #10](../part5/gotchas.md#10-rest-account-flag-is---name-not---name-account).

---

## PDA Seed Formats

The `lez-cli pda` command accepts seeds as a JSON array string:

```bash
lez-cli pda \
  --seeds '["seed_string", "another_string"]' \
  --program-id <image-id>
```

### Seed Types in JSON

| Seed Content | JSON Format | Example |
|--------------|------------|---------|
| String literal | JSON string | `"state"` |
| Byte array | JSON array of integers | `[1, 2, 3, 4]` |
| Hex bytes | JSON string with hex encoding (check version) | `"0102030405"` |

**Full example:**

```bash
# Single string seed
lez-cli pda --seeds '["global_state"]' --program-id <id>

# Multiple seeds
lez-cli pda --seeds '["user", "5Kg8cf..."]' --program-id <id>

# String + bytes
lez-cli pda --seeds '["vault", [1, 0, 0, 0]]' --program-id <id>
```

> **⚠️ Warning:** `lez-cli pda` produces incorrect addresses for `u64` and `u128` integer seeds. Use string seeds when possible, or derive PDA addresses from sequencer logs instead of the CLI. See [Gotchas #3](../part5/gotchas.md#3-lez-cli-pda-gives-wrong-addresses-for-u64u128-seeds).

---

## Compound Types

For complex types (structs, enums) exposed as instruction arguments, check your IDL to see how they are flattened. LEZ typically flattens nested structs into individual CLI flags using dotted notation or positional encoding. Refer to the generated IDL for the exact flattened form of each argument.

---

## Shell Quoting Notes

When passing values on the command line, follow standard shell quoting rules:

```bash
# String with spaces — quote the whole value
--name "hello world"

# JSON seeds — use single quotes to avoid shell escaping
--seeds '["state", "user"]'

# Hex strings — no special quoting needed
--hash deadbeef0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c

# Large integers — no quoting needed
--amount 18446744073709551615

# Boolean — no quoting needed
--is-active true
```

---

## Type Reference in IDL JSON

The generated IDL JSON uses the following type identifiers:

| IDL Type String | LEZ Type | CLI Format |
|----------------|----------|-----------|
| `"u8"` | `u8` | decimal |
| `"u32"` | `u32` | decimal |
| `"u64"` | `u64` | decimal |
| `"u128"` | `u128` | decimal |
| `"bool"` | `bool` | `true`/`false` |
| `"string"` | `String` | bare string |
| `{"array": ["u8", 32]}` | `[u8; 32]` | hex string |
| `{"vec": "u8"}` | `Vec<u8>` | comma-separated decimal |
| `"publicKey"` | `AccountId` | base58 |

Use the IDL type string to determine which CLI format to use when the type is not obvious from the argument name.

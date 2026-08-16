# Booth Checker — Proof of Concept

**Flag:** `UNI6CTF{Az4d@Vector_9mK!40}`

## Overview

`booth_checker.py` reads a custom binary pass format identified by the `LBTH` header. The parser reconstructs a 128-byte memory area and chooses one of four actions using a byte stored in that memory:

```text
0 → preview
1 → deny
2 → audit
3 → reveal
```

The goal is to construct a valid pass that reaches `reveal()` with the required state.

## 1. Reveal Conditions

The final routine requires:

```text
mem[32:40]   == b"AUG15"
mem[112] % 4 == 3
mem[113:121] == SHA256(b"BOOTH|15-08-1947")[:8]
```

The authentication token is deterministic:

```python
import hashlib

DATE = b"15-08-1947"
token = hashlib.sha256(b"BOOTH|" + DATE).digest()[:8]
```

## 2. Useful Record Type

The format has three record types:

```text
01 → write to mem[0:32]
02 → write to mem[32:40]
03 → patch memory using an offset relative to base 48
```

Record `03` is the important primitive. Its first data byte supplies the offset and the remaining bytes are copied into memory.

An offset of `64` therefore targets:

```text
48 + 64 = 112
```

So one patch can establish both the action selector and the token:

```python
patch_data = bytes([64, 3]) + token
```

## 3. Create the Forged Pass

```python
def rec(record_type, data):
    return bytes([record_type, len(data)]) + data

zone_rec = rec(2, b"AUG15")

patch_data = bytes([64, 3]) + token
patch_rec = rec(3, patch_data)

count = 2

blob = (
    b"LBTH"
    + bytes([count])
    + zone_rec
    + patch_rec
)

with open("mypass.bin", "wb") as f:
    f.write(blob)
```

This produces the required memory state:

```text
mem[32:40]   = AUG15
mem[112]     = 3
mem[113:121] = expected token
```

## 4. Run the Checker

```bash
python3 booth_checker.py mypass.bin
```

The reveal branch is reached and the embedded encrypted value is recovered.

## Result

```text
UNI6CTF{Az4d@Vector_9mK!40}
```

## Key Point

The custom file format provides enough write capability to configure fields that ordinary records cannot reach. Once the patch record is used to set the dispatch byte and SHA-256-derived token, the reveal condition is satisfied.

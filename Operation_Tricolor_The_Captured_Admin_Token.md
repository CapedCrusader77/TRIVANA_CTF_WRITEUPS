# Operation Tricolor: The Captured Admin Token

**Category:** Crypto
**Difficulty:** Easy
**Points:** 150
**Platform:** CSEMA — Cyber Security Practice Lab

---

## Challenge Description

The CSEMA Security Operations Team is investigating suspicious activity inside a simulated government network. An internal authentication service has been protecting session tokens with a repeating key cipher. An administrative session token was captured in the organization's proxy logs. The encryption relies on a repeating key, making the ciphertext vulnerable to cryptanalysis. The mission is to investigate the proxy evidence, determine the repeating-key pattern, and recover the hidden administrative credential.

**Files provided:**
- `auth_proxy_traffic.log`
- `session_vault.json`

---

## Flag

```
TRIVARNA{xor_repeating_key_crib_drag_auth_proxy_2026}
```

> **Dynamic flag** — the inner value is recovered by XOR-decrypting the captured hex token with key `PROXYKEY2026`. Wrap whatever you recover in `TRIVARNA{...}`.

---

## Solution

### Step 1 — Find the Evidence

Both files are large noise logs. Grep for relevant keywords in each:

```bash
# Proxy log
grep -n "admin\|encrypt\|token\|hex\|xor\|key" auth_proxy_traffic.log -i

# Session vault
grep -n "admin\|encrypt\|xor\|key" session_vault.json -i
```

**Proxy log — lines 1201 and 1203:**

```
[WARN] [AUTH-PROXY] Intercepted admin session hex token:
363e2e3f22332a2b6d42574635333b31372c1a3257496d

[INFO] [AUTH-PROXY] Token encryption scheme:
Repeating-Key XOR (Key ID: PROXYKEY2026)
```

**Session vault — line 1451 (entry `ADMIN-CRIT-99`):**

```json
{
  "session_id": "ADMIN-CRIT-99",
  "user": "admin@core.jio.com",
  "encryption": "REPEATING_XOR",
  "key_ascii": "PROXYKEY2026",
  "encrypted_token_hex": "363e2e3f22332a2b6d42574635333b31372c1a3257496d55223b2d073d39243e6d514742380d3f2a36333c06000000002d"
}
```

> **Critical flaw:** The XOR key `PROXYKEY2026` is stored in plain text in the same JSON record as the ciphertext. This completely negates any security the encryption might otherwise provide.

---

### Step 2 — Understanding Repeating-Key XOR

Repeating-key XOR encrypts by XORing each plaintext byte with the corresponding byte of a key that cycles repeatedly:

```
Plaintext:  f  l  a  g  {  x  o  r  _  r  e  p  ...
Key:        P  R  O  X  Y  K  E  Y  2  0  2  6  P  R  O  X  ...  (repeats every 12 bytes)
            ↕  ↕  ↕  ↕  ↕  ↕  ↕  ↕  ↕  ↕  ↕  ↕
Ciphertext: 36 3e 2e 3f 22 33 2a 2b 6d 42 57 46  ...  (hex)
```

Decryption formula:

```
plaintext[i] = ciphertext[i] XOR key[i % key_length]
```

XOR is its own inverse — encryption and decryption are the same operation.

---

### Step 3 — Decrypt the Token

```python
# Values from session_vault.json ADMIN-CRIT-99
ciphertext_hex = (
    "363e2e3f22332a2b6d42574635333b31372c1a3257496d"
    "55223b2d073d39243e6d514742380d3f2a36333c06000000002d"
)
key = "PROXYKEY2026"

ct = bytes.fromhex(ciphertext_hex)
k  = key.encode()

plaintext = bytes([ct[i] ^ k[i % len(k)] for i in range(len(ct))])
print(plaintext.decode())
# → flag{xor_repeating_key_crib_drag_auth_proxy_2026}
```

**Decryption summary:**

| Field | Value |
|-------|-------|
| Key | `PROXYKEY2026` (12 bytes) |
| Ciphertext (hex) | `363e2e3f22332a2b6d42574635333b31372c1a3257496d55223b2d073d39243e6d514742380d3f2a36333c06000000002d` |
| Plaintext | `flag{xor_repeating_key_crib_drag_auth_proxy_2026}` |
| Length | 49 bytes (flag + null padding) |

---

### Bonus — Crib Drag Attack (if key was unknown)

The challenge name hints at **crib dragging** — the classical attack on repeating-key XOR when the key is not known. It works because:

```
If two messages share the same key:
  C1 XOR C2 = P1 XOR P2
```

The key cancels out, leaving the XOR of the two plaintexts. A known plaintext guess ("crib") is then slid across this value to recover key bytes at each offset:

```python
def crib_drag(c1, c2, crib):
    for offset in range(len(c1) - len(crib)):
        segment  = bytes([c1[offset+i] ^ c2[offset+i] for i in range(len(crib))])
        candidate = bytes([segment[i] ^ crib[i] for i in range(len(crib))])
        if candidate.isascii() and candidate.isprintable():
            print(f"offset={offset}: {candidate}")
```

Common cribs for session tokens: `flag{`, `admin`, `user=`, `token`. In this challenge the key was leaked alongside the ciphertext, so no crib dragging was required — the title hints at the class of attack, not a required step.

---

## Key Takeaways

- Storing the encryption key next to the ciphertext provides **zero security** — it's equivalent to no encryption at all.
- Repeating-key XOR is a **broken cipher**; even without the key leak it is vulnerable to crib dragging whenever two or more messages share the same key.
- The flag was recoverable with a single line of Python once the key was found.

---

## Mitigations

- **Never store keys with ciphertext** — keys must be managed separately (e.g., a hardware security module or secrets manager).
- **Use authenticated encryption** — replace XOR with AES-GCM or ChaCha20-Poly1305, which resist key-reuse and tampering.
- **Unique key per session** — generate a cryptographically random key or nonce for every new session token.
- **Prefer signed tokens** — for session authentication, signed tokens (e.g., JWT with RS256) are often more appropriate than encrypted ones.

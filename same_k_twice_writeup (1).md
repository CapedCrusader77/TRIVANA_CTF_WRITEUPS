# Same k, Twice — Writeup

**Category:** Crypto | **Difficulty:** Hard | **Points:** 250

## TL;DR

Two ECDSA signatures reused the same nonce `k` (same `r`), which leaks the
private key with a couple lines of modular arithmetic. From there, a
deterministic-nonce "challenge" scheme lets us reproduce a signature on a
new message, and a hash of that signature's `s` value turns out to be the
AES key protecting an encrypted export blob.

```
CSEMA{r3us3d_n0nc3_l34ks_pr1v4t3_k3y}
```

Submitted (per the event's wrapper convention) as:

```
TRIVARNA{CSEMA{r3us3d_n0nc3_l34ks_pr1v4t3_k3y}}
```

---

## 1. Given

- Curve: **secp256k1**, with explicit `P`, `A=0`, `B=7`, `N`, generator `G`.
- A public key `Q`.
- Two ECDSA signatures over two different messages.
- A `challenge_message` and a description of a **deterministic nonce
  scheme**: `k = int(SHA256(priv_bytes_32BE || message)) mod N`.
- An AES-CBC `export_blob` (`iv` + `ciphertext`) to decrypt.

## 2. Sanity-check the curve

The prompt hints "verify the generator point is actually on the curve
before using it" — so first confirm both `G` and `Q` satisfy
`y² ≡ x³ + 7 (mod P)`:

```python
lhs = (Gy*Gy) % P
rhs = (Gx**3 + A*Gx + B) % P
assert lhs == rhs        # True

lhsQ = (Qy*Qy) % P
rhsQ = (Qx**3 + A*Qx + B) % P
assert lhsQ == rhsQ      # True
```

Both check out — standard secp256k1 params, no trickery there.

## 3. Spot the nonce reuse

Looking at the two signatures:

```
sig1: r = 0x67a7164f...b82cde   s = 0x85a03b1b...77eb19a
sig2: r = 0x67a7164f...b82cde   s = 0xab094dc0...28d0fdb
```

**Identical `r`.** Since `r` is `(k·G).x mod N`, an identical `r` across two
signatures (with overwhelming probability) means the **same nonce `k`** was
used for both. That's a textbook ECDSA nonce-reuse vulnerability.

## 4. Recover k and the private key

For ECDSA, `s = k⁻¹(z + r·priv) mod N`, where `z` is the hash of the
message. With two signatures sharing `k` and `r`:

```
s1 - s2 = k⁻¹(z1 - z2)  (mod N)
  ⇒  k = (z1 - z2) / (s1 - s2)  mod N
```

Then recover the private key from either signature:

```
priv = (s1·k - z1) / r  mod N
```

```python
z1 = sha256_int(m1.encode())
z2 = sha256_int(m2.encode())

k    = ((z1 - z2) * pow((s1 - s2) % N, -1, N)) % N
priv = ((s1*k - z1) * pow(r % N, -1, N)) % N
```

Result:

```
k    = 0x781b9a43d04ce50b0620f0877e5fe38183faac572f564652466de486522c4f8e
priv = 0x6102dd7063e8540e9dd8904f0748967121bade026a6ae768f2ed66ffdcc99397
```

**Verification:** computed `priv * G` with a hand-rolled EC point
add/double (no external ECC library needed) and confirmed it equals the
given public key `Q`. ✅

## 5. Reproduce the "challenge" signature

The challenge describes a *deterministic* nonce derivation:

```
k_chal = int(SHA256(priv_bytes_32BE || challenge_message)) mod N
```

Since we now hold `priv`, we can compute this exactly as the signer would:

```python
priv_bytes = priv.to_bytes(32, 'big')
k_chal = int.from_bytes(
    hashlib.sha256(priv_bytes + challenge_message.encode()).digest(), 'big'
) % N
```

Then produce the full ECDSA signature on `challenge_message =
"UNLOCK-EXPORT-CSEM-99"` using this `k_chal` and our recovered `priv`:

```python
R = k_chal * G
r_chal = R.x % N
z_chal = sha256_int(challenge_message.encode())
s_chal = (pow(k_chal, -1, N) * (z_chal + r_chal * priv)) % N
```

## 6. Crack the export blob

The `export_blob` is AES-CBC (`iv` + `ciphertext`, 48 bytes → 3 blocks,
PKCS7 padding expected). The key isn't given directly, so I brute-forced a
handful of natural candidates derived from the values already computed:
`priv`, `k_chal`, `r_chal`, `s_chal`, each raw and as a SHA-256 digest, at
both 16 and 32 bytes (AES-128 / AES-256).

The hit:

```
key = SHA256(s_chal.to_bytes(32, 'big'))[:16]     # AES-128-CBC
iv  = given iv
```

```python
cipher = Cipher(algorithms.AES(key), modes.CBC(iv))
pt = cipher.decryptor().update(ciphertext) + cipher.decryptor().finalize()
```

This decrypts cleanly with valid PKCS7 padding (`\x0b` × 11) to:

```
CSEMA{r3us3d_n0nc3_l34ks_pr1v4t3_k3y}
```

## 7. Flag

Per the event's stated wrapper convention (`TRIVARNA{...}`, "submit the
full string including the wrapper"):

```
TRIVARNA{CSEMA{r3us3d_n0nc3_l34ks_pr1v4t3_k3y}}
```

---

## Root cause / lesson

ECDSA's security depends entirely on the nonce `k` being **unique and
unpredictable** per signature. Reusing `k` across two signatures — even
with different messages — turns two linear equations in two unknowns
(`k`, `priv`) into a trivially solvable system. This is the same class of
bug that famously leaked the PS3's signing key. The fix is to always use a
fresh random nonce per signature, or a properly domain-separated
deterministic scheme like **RFC 6979**, which binds the nonce to both the
private key *and* the full message hash in a way that's collision-resistant
by construction — unlike the toy `SHA256(priv || message)` scheme used
here, which (ironically) was still safe against reuse *for distinct
messages*, but was leveraged here simply because it was *reproducible*
once we already had `priv`, letting us regenerate the "secret" derived
key.

## Full solve script

```python
import hashlib
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

P = int("0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f", 16)
A, B = 0, 7
N = int("0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141", 16)
Gx = int("0x79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798", 16)
Gy = int("0x483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8", 16)
Qx = int("0x9228026a997ebd23250c383fab6277feb034115a880a7030b75aec301fcd067e", 16)
Qy = int("0xd27f170d9cea2db5166b079184a36c1db23fbbb77965443d288c389fc05ad525", 16)

def inv(a, m): return pow(a, -1, m)
def sha256int(b): return int.from_bytes(hashlib.sha256(b).digest(), 'big')

def point_add(p1, p2):
    if p1 is None: return p2
    if p2 is None: return p1
    x1, y1 = p1; x2, y2 = p2
    if x1 == x2 and (y1 + y2) % P == 0: return None
    lam = (3*x1*x1 + A) * inv(2*y1, P) % P if p1 == p2 else (y2 - y1) * inv((x2 - x1) % P, P) % P
    x3 = (lam*lam - x1 - x2) % P
    y3 = (lam*(x1 - x3) - y1) % P
    return (x3, y3)

def scalar_mult(k, point):
    result, addend = None, point
    while k:
        if k & 1: result = point_add(result, addend)
        addend = point_add(addend, addend)
        k >>= 1
    return result

# --- Step 1: curve sanity checks ---
assert (Gy*Gy) % P == (Gx**3 + A*Gx + B) % P
assert (Qy*Qy) % P == (Qx**3 + A*Qx + B) % P

# --- Step 2: nonce-reuse key recovery ---
m1 = "AUTH-LOG:2026-08-15T09:00:00Z:hub-04:pairing-approved"
m2 = "AUTH-LOG:2026-08-15T09:04:00Z:hub-07:pairing-approved"
r  = int("0x67a7164f30ec44f0bf61da763122f7e43d153789abaa32ee12e5154da1b82cde", 16)
s1 = int("0x85a03b1b32769b9a2460694c586b40ba0ee8d2c47839295620c35c00677eb19a", 16)
s2 = int("0xab094dc072fda3ad3a93b6b487656daac1e5b59d72b2d8f68776b59c428d0fdb", 16)

z1, z2 = sha256int(m1.encode()), sha256int(m2.encode())
k    = ((z1 - z2) * inv((s1 - s2) % N, N)) % N
priv = ((s1*k - z1) * inv(r % N, N)) % N
assert scalar_mult(priv, (Gx, Gy)) == (Qx, Qy)

# --- Step 3: deterministic nonce for the challenge message ---
challenge_message = "UNLOCK-EXPORT-CSEM-99"
priv_bytes = priv.to_bytes(32, 'big')
k_chal = sha256int(priv_bytes + challenge_message.encode()) % N

Rp = scalar_mult(k_chal, (Gx, Gy))
r_chal = Rp[0] % N
z_chal = sha256int(challenge_message.encode())
s_chal = (inv(k_chal, N) * (z_chal + r_chal*priv)) % N

# --- Step 4: AES-128-CBC decrypt with key = SHA256(s_chal)[:16] ---
iv  = bytes.fromhex("608c65b8340bfc3fb10dd569052a772e")
ct  = bytes.fromhex("9cf992082f41fab8b2e49f5f8b4de7330b6b017efb2babc6a4cea552549def2d58f2f5ccf25f6d32ea88209495bde7fc")
key = hashlib.sha256(s_chal.to_bytes(32, 'big')).digest()[:16]

dec = Cipher(algorithms.AES(key), modes.CBC(iv)).decryptor()
pt  = dec.update(ct) + dec.finalize()
print(pt)   # b'CSEMA{r3us3d_n0nc3_l34ks_pr1v4t3_k3y}\x0b\x0b...'
```

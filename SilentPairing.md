# Silent Pairing

**Category:** Crypto / IoT / Hardware  
**Difficulty:** Easy  
**Platform:** UNI6CTF / Trivarna

---

## Challenge Overview

The challenge provides a synthetic BLE capture from a smart-device hub. The goal is to identify the device involved in the actual pairing handshake, recover the weak pairing secret, and use it to decrypt an encrypted OTA packet.

The important observation is that the pairing protocol uses predictable values instead of proper randomness.

---

## 1. Identify the Target Device

The capture contains several `PAIR_INIT`, `PAIR_ACK`, and `OTA_CHUNK` packets.

The three OTA destinations are:

```text
C7:93:31:48:D4:FA
D5:21:A8:29:00:16
04:5D:4F:F5:59:96
```

Only one of these devices has both a complete pairing handshake and an OTA packet.

The relevant device is:

```text
D5:21:A8:29:00:16
```

Its handshake is:

```text
PAIR_INIT
nonce = 1755000314
```

followed by:

```text
PAIR_ACK
ack_nonce = 3443427423
```

and the OTA packet is sent to the same device.

---

## 2. Analyze the Pairing Values

The first important weakness is that the `nonce` is not actually random.

For the target device:

```text
nonce = 1755000314
timestamp = 1755000314
```

The nonce is exactly equal to the packet timestamp.

So the supposedly random value can be predicted from the capture timestamp.

The second observation is:

```text
ack_nonce = nonce XOR 0xA5A5A5A5
```

This relationship also holds for the other pairing exchanges in the capture.

Therefore the pairing handshake does not provide meaningful cryptographic entropy.

---

## 3. Derive the AES Key

The OTA packet uses AES-128-CBC.

The target packet contains:

```text
IV:
ef42fe0a3322aee5043a2ca3b1a6fc80
```

The useful pairing value is the predictable nonce:

```text
1755000314
```

Testing the nonce-derived key construction reveals that the challenge uses:

```text
MD5(str(nonce))
```

So the AES-128 key is:

```text
MD5("1755000314")
```

This produces a valid 16-byte AES key.

---

## 4. Decrypt the OTA Packet

The target OTA ciphertext is:

```text
a9b1743472d8a2ff486879331a1eeabc5c8b7c5903d66f147092daa66d58b300f345b4c0c05178f32fd34049c0d02aa21aa92d9302da17fe598fa4a46a24164a0e316b07d708804edbcc22072ee815fcc601caf2015dfbc96ad5b1a536f3f719365915cbebd88c61bc8b60cbb90d21fa
```

Using:

```text
AES-128-CBC
Key = MD5("1755000314")
IV  = ef42fe0a3322aee5043a2ca3b1a6fc80
```

the decrypted data contains a reversed readable section.

The meaningful portion is:

```text
RTFMCSEMA{n0nc3_reuse_1s_n3v3r_0k_4y}
```

Reversing the encoded wrapper gives the flag body:

```text
n0nc3_reuse_1s_n3v3r_0k_4y
```

The intended submission format is:

```text
TRIVARNA{n0nc3_reuse_1s_n3v3r_0k_4y}
```

---

## 5. Complete Solve Chain

```text
BLE capture
     ↓
Find OTA_CHUNK packets
     ↓
Match OTA destination with PAIR_INIT / PAIR_ACK
     ↓
Target = D5:21:A8:29:00:16
     ↓
Recover nonce = 1755000314
     ↓
Notice nonce == timestamp
     ↓
MD5(str(nonce))
     ↓
AES-128-CBC
     ↓
IV = ef42fe0a3322aee5043a2ca3b1a6fc80
     ↓
Decrypt OTA payload
     ↓
Reverse the meaningful encoded section
     ↓
n0nc3_reuse_1s_n3v3r_0k_4y
     ↓
TRIVARNA{n0nc3_reuse_1s_n3v3r_0k_4y}
```

---

## Flag

```text
TRIVARNA{n0nc3_reuse_1s_n3v3r_0k_4y}
```

---

## Key Takeaways

- Never assume a protocol nonce is random just because it is called a nonce.
- Comparing a nonce with packet timestamps can reveal predictable RNG mistakes.
- A fixed transformation such as `nonce XOR 0xA5A5A5A5` provides no real entropy.
- Matching the pairing handshake to the OTA destination is essential when multiple devices are present.
- Once the predictable nonce was identified, the MD5-derived AES key allowed the OTA payload to be decrypted.
- The challenge is ultimately about recognizing **nonce reuse / predictable nonce generation** as the core weakness.

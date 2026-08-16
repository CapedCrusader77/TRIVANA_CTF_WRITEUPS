# CTF Writeup — Cipher Cascade

**Event:** TRIVARNA 2.0 (CSEMA)
**Challenge:** Cipher Cascade
**Category:** Cryptography / Encoding
**Difficulty:** Medium
**Points:** 200
**Solves:** 1

---

## Challenge Description

> A threat actor believed that applying multiple encoding and transformation techniques would make their secret impossible to recover. Instead of relying on a single encryption algorithm, they chained together several common and uncommon encodings, hoping that the complexity alone would discourage anyone from investigating further. During a forensic investigation, analysts recovered the following encoded text from the attacker's workstation. The original message still exists beneath the layers—you simply need to remove each one in the correct order.

**Ciphertext:**

```
4BjN/%1q?z[0)FX4<o%2Bd^bEQXCVxZqAH;XFq]NPX]5lb%Y;Itto90,maF#&ij>:%:p:E[$9[P[UC9L9a$E8w\Xg^H6@AXJ4g<ED6+#.uHs)_Ab5+%F4d+2EfURVx]|A#uLk--\J{_Dn!ai=3]9lJ1a/'bJ?fRZ?YFh#t.dJqEDmVGA>4QFE]^S[I[tViZn:$%R-e\8.JZM?=OW5,120U+D)S]>&P3XA&%o,@XU#bY'=*ar<n<Ln!12#pF&UlOl@aub:8YimN
```

---

## Initial Analysis

The blob is **266 characters** long and uses **83 distinct printable ASCII characters** spanning codes 33–124.

The first instinct is to try the usual suspects: Base64, Base32, hex, URL-decode, Base85 variants (Ascii85, RFC 1924, Z85), and Base91. All fail immediately — each encoding has a narrower alphabet, and a handful of different characters in the blob break each one.

Running CyberChef's **Magic** operation (depth 4, intensive) returns only EBCDIC and UTF-16 noise. That is actually informative: Magic doesn't carry Base92 in its operation set, so finding nothing clean is a strong hint that Base92 is involved.

---

## The Encoding Stack

There are **five layers**, applied from innermost outward (so peeled in this order):

```
Ciphertext
  └─ Layer 1: Base92
       └─ Layer 2: Base32
            └─ Layer 3: Unicode shift (U+7C00 block)
                 └─ Layer 4: ROT13
                      └─ Layer 5: Base64
                           └─ FLAG
```

---

## Layer 1 — Base92

**Base92** encodes binary as printable ASCII using every character in the range `!` (0x21) through `}` (0x7D) *except* `"` (0x22) and `` ` `` (0x60), giving exactly 92 symbols. It packs **13 bits per character pair**, which means the blob length has no clean relationship to standard 4- or 5-character group sizes — another reason automated tools miss it.

The full character set of the blob (`!#$%&')*+,-./0-9:;<=>?@A-Z[\]^_a-z{|}`) is a perfect match for the Base92 alphabet.

Decoding yields a Base32-encoded string:

```
46ZIDZ5SQDT3DLPHWCY6PMNZ46YZFZ5RXHT3FAHHWG66PMF346YYDZ5QWDT3BNHHWCX6PMML...====
```

---

## Layer 2 — Base32

Straightforward standard Base32. The output is 44 **CJK Unified Ideographs** in the range U+7C2B–U+7C87:

```
粁粀籭簱籹籒籹粀籽簻籁簰簴簯籋籍籽簫粇籍粂籿粇籩粃籨籁籂簴簯籋籍籽粃籏籄簴粂籱籨簴籫簱籱
```

This is designed to look like legitimate Chinese text and fool character-frequency analysis.

---

## Layer 3 — Unicode Shift (U+7C00 block)

Each CJK codepoint encodes a printable ASCII character via the mapping:

```
encoded_codepoint = U+7C00 + ((ascii_value - 38) mod 94)
```

Reversing it:

```
ascii_value = 33 + ((codepoint - 0x7C00 + 5) mod 94)
```

The correct offset (5) is found by testing all 94 shifts and checking which yields a fully alphanumeric string. Only `+5` works:

```
IH5WAxAHEagVZUqsEQOsJGO1K0ghZUqsEKujZJ90Z3W9
```

---

## Layer 4 — ROT13

This is the sneakiest layer. The string coming out of Layer 3 *looks* like valid Base64 — all alphanumeric with no padding issues. Decoding it directly gives 33 bytes of high-entropy binary. The correct step is to apply **ROT13 first**:

```
IH5WAxAHEagVZUqsEQOsJGO1K0ghZUqsEKujZJ90Z3W9
  ──ROT13──▶
VU5JNkNURntIMHdfRDBfWTB1X0tuMHdfRXhwMW90M3J9
```

The repeated `ZUqsE` pattern at positions 12 and 28 in the pre-ROT13 string was a deliberate distraction — it corresponds to `_D0_` and `_Kn0` in the final plaintext landing on the same Base64 4-character boundary.

---

## Layer 5 — Base64

Standard Base64 decode of the ROT13 output:

```
VU5JNkNURntIMHdfRDBfWTB1X0tuMHdfRXhwMW90M3J9
  ──Base64──▶
UNI6CTF{H0w_D0_Y0u_Kn0w_Exp1ot3r}
```

---

## Flag

```
UNI6CTF{H0w_D0_Y0u_Kn0w_Exp1ot3r}
```

---

## Solve Script

```python
import base64

S = "4BjN/%1q?z[0)FX4<o%2Bd^bEQXCVxZqAH;XFq]NPX]5lb%Y;Itto90,maF#&ij>:%:p:E[$9[P[UC9L9a$E8w\\Xg^H6@AXJ4g<ED6+#.uHs)_Ab5+%F4d+2EfURVx]|A#uLk--\\J{_Dn!ai=3]9lJ1a/'bJ?fRZ?YFh#t.dJqEDmVGA>4QFE]^S[I[tViZn:$%R-e\\8.JZM?=OW5,120U+D)S]>&P3XA&%o,@XU#bY'=*ar<n<Ln!12#pF&UlOl@aub:8YimN"

# Layer 1: Base92 decode
def b92ord(ch):
    o = ord(ch)
    if ch == '!': return 0
    if 35 <= o <= 95: return o - 34
    if 97 <= o <= 125: return o - 35
    raise ValueError(ch)

def b92decode(s):
    bits = ''; out = bytearray()
    for i in range(0, len(s) - 1, 2):
        bits += format(b92ord(s[i]) * 91 + b92ord(s[i+1]), '013b')
        while len(bits) >= 8:
            out.append(int(bits[:8], 2)); bits = bits[8:]
    if len(s) % 2:
        bits += format(b92ord(s[-1]), '06b')
        while len(bits) >= 8:
            out.append(int(bits[:8], 2)); bits = bits[8:]
    return bytes(out)

L1 = b92decode(S).decode()                         # → Base32 string
L2 = base64.b32decode(L1).decode('utf-8')          # → CJK characters
L3 = ''.join(chr(33 + ((ord(c) - 0x7C00 + 5) % 94)) for c in L2)   # → alnum
L4 = ''.join(                                       # ROT13
    chr((ord(c) - 65 + 13) % 26 + 65) if c.isupper() else
    chr((ord(c) - 97 + 13) % 26 + 97) if c.islower() else c
    for c in L3)
FLAG = base64.b64decode(L4 + '==').decode()        # → flag

print(FLAG)
# UNI6CTF{H0w_D0_Y0u_Kn0w_Exp1ot3r}
```

---

## Key Takeaways

- **Base92 defeats CyberChef Magic.** Neither Base92 nor Base122 appear in Magic's operation list, so a blob using nearly all printable ASCII should put these at the top of the manual checklist.
- **"Looks like Base64" is a trap.** An alphanumeric string that decodes cleanly may still have an outer transform. If the decode yields high-entropy binary, try ROT13/ROT47/Caesar shifts on the encoded form first.
- **CJK output from Base32 is intentional camouflage.** The range U+7C00–U+7C87 falls squarely in CJK Unified Ideographs; a character-frequency tool will tag it as Chinese text and send analysts down the wrong path.
- **Shift identification:** testing all 94 ASCII-94 shifts and filtering for a fully alphanumeric output uniquely identifies the correct offset in under a second.

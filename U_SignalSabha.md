# Signal Sabha

**Category:** Misc / Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Platform:** UNI6CTF / Trivarna

---

## Challenge Overview

The challenge contains relay logs with a mixture of valid records, fake flags, wrong dates, and timestamps from different time zones.

The goal is to identify the genuine ceremony records, decode their fragments, order them correctly, and then apply the final transformation to recover the flag.

---

## 1. Identify the Useful Records

The challenge description provides several clues:

- real tricolor fragments
- wrong-date rehearsals
- fake flags
- timestamps from different time zones
- mixed-base encoding
- fragments logged out of order

The genuine records match the Independence Day ceremony conditions:

```text
date  = 15-08-1947
tag   = CEREMONY
valid = true
```

Records that do not satisfy these conditions are treated as decoys.

---

## 2. Decode the Fragments

The valid records use different encodings depending on their tricolor section.

The encoding mapping is:

```text
saffron → hexadecimal
white   → Base64
green   → Base32
```

After decoding the individual fragments, they still need to be placed in the correct chronological order.

---

## 3. Correct the Timestamp Order

The logs contain timestamps from different time zones.

Because of this, sorting the displayed timestamps directly can produce the wrong sequence.

The timestamps first need to be normalized to a common timezone, specifically UTC, and then sorted chronologically.

After ordering the decoded fragments, the intermediate string becomes:

```text
UICFSR@rba_u!3N6T{aePaht4M6}
```

This is clearly not the final flag, so another transformation is required.

---

## 4. Recognize the Final Transformation

The challenge indicates that the final message was split and logged out of order.

The intermediate string has the structure of a **2-rail Rail Fence transformation**.

Applying a Rail Fence decryption with 2 rails produces:

```text
UNI6CTF{SaRe@Prabhat_4uM!63}
```

This gives the intended flag body.

---

## 5. Recover the Final Flag

The CTF submission system uses the `TRIVARNA{}` wrapper.

Therefore the final submission is:

```text
TRIVARNA{SaRe@Prabhat_4uM!63}
```

---

## Complete Solve Chain

```text
Relay logs
    ↓
Filter valid CEREMONY records
    ↓
Keep records dated 15-08-1947
    ↓
Decode tricolor fragments
    ↓
Saffron → Hex
White   → Base64
Green   → Base32
    ↓
Normalize timestamps to UTC
    ↓
Sort fragments chronologically
    ↓
UICFSR@rba_u!3N6T{aePaht4M6}
    ↓
2-rail Rail Fence decode
    ↓
UNI6CTF{SaRe@Prabhat_4uM!63}
    ↓
TRIVARNA{SaRe@Prabhat_4uM!63}
```

---

## Flag

```text
TRIVARNA{SaRe@Prabhat_4uM!63}
```

---

## Key Takeaways

- Filter the genuine records before decoding anything.
- Mixed encodings can be used for different fragments of the same message.
- Timestamps from different time zones must be normalized before chronological sorting.
- A readable intermediate string does not necessarily mean the challenge is finished.
- The final 2-rail Rail Fence transformation reveals the actual flag.

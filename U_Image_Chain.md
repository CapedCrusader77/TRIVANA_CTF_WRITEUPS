# U - Image Chain

**Challenge:** U - Image Chain  
**Platform:** UNI6CTF / Trivarna  
**Category:** Crypto / OSINT  
**Difficulty:** Hard

## Objective

Recover the hidden destination from the supplied encoded string, use that destination to continue the public clue trail, and obtain the challenge flag.

## Reconnaissance

The initial artifact looked like a malformed web address rather than conventional ciphertext:

```text
sggkh://ulinh.tov/h9UACJvFWtiZecK26
```

The unusual characters in the protocol and domain were the strongest hint. Instead of attempting to brute-force the path or treat the string as a hash, the obvious first test was a letter-for-letter alphabet reversal.

## Decoding the First Layer

Use the standard Atbash mapping:

- `A ↔ Z`
- `B ↔ Y`
- ...
- `a ↔ z`
- `b ↔ y`
- ...

Applying it across the complete string transforms the URL into a valid Google Forms address.

The important detail is that the decoded path/short-code must be copied **exactly as produced**. Changing capitalization in the code can send the investigation to a different destination.

## Following the Chain

The decoded Google Form was not the flag itself. It acted as the next link in the challenge.

Opening the form and examining the supplied response revealed the final flag value. There was no need for:

- password guessing
- brute-force searching
- steganographic extraction
- image manipulation
- cryptographic cracking beyond the Atbash substitution

The challenge therefore consists of a lightweight substitution layer followed by an OSINT-style public-resource lookup.

## Solve Flow

```text
Encoded URL
    │
    ▼
Atbash substitution
    │
    ▼
Valid Google Forms URL
    │
    ▼
Inspect public form response
    │
    ▼
Recover exact flag
```

## Flag

```text
UNI6CTF{Y0u_m5s1e7_im5g2_c1p8e7s}
```

## Takeaways

1. A broken-looking URL can itself be the ciphertext.
2. When the protocol and hostname both look systematically altered, try simple substitution ciphers before more complicated attacks.
3. Once a URL becomes valid, treat the destination as part of the puzzle rather than assuming the decoded text is the answer.
4. Preserve capitalization and leetspeak exactly when copying a CTF flag.

## Final Conclusion

The intended path was straightforward once the substitution was recognized: reverse the alphabet with Atbash, follow the resulting Google Forms link, and extract the flag from the public response. The challenge combines a basic classical cipher with a short OSINT chain, making the initial URL recognition the key breakthrough.

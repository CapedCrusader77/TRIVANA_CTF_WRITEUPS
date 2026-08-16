# New Destiny — Reverse Engineering Writeup

**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 350

## Flag

```text
TRIVARNA{SaRe_Jahan#1947_xYz!}
```

## 1. First Look

The challenge provides a single unusual file:

```text
verifier.blob
```

The supplied hint is:

> “No shortcuts, no keystreams — just a matrix and a stretch of time between you and the destiny.”

That strongly suggests that the interesting logic is hidden inside a verifier rather than exposed directly.

A normal file inspection does not immediately identify an executable:

```bash
file verifier.blob
strings verifier.blob | head
```

The contents instead look like encoded container data.

## 2. Unpacking the Blob

Inspection of the beginning of the file reveals a Base64-like representation.

After decoding that layer, the output is compressed. Repeating the appropriate decompression operations reveals several nested layers:

```text
verifier.blob
      |
      v
   Base64
      |
      v
      XZ
      |
      v
    BZip2
      |
      v
     GZip
      |
      v
  verifier.js
```

The important point is that the apparent blob is only a wrapper. The actual challenge logic is in the recovered JavaScript.

## 3. Inspecting `verifier.js`

The JavaScript is intentionally difficult to read.

Some characteristics include:

- meaningless variable names,
- large numeric arrays,
- arithmetic transformations,
- indirect string construction,
- an input verification routine.

Two structures are especially relevant:

```text
_MFI
_FCT
```

The matrix hinted at by the challenge description appears in `_MFI`.

## 4. Following the Verification Routine

The verifier does not compare the submitted string directly with a plaintext flag.

Instead, the supplied input passes through a transformation resembling:

```text
user input
    |
    v
matrix operation
    |
    v
modulo 257
    |
    v
comparison with stored values
```

The transformed result is checked against constants contained in `_FCT`.

A complete mathematical reversal of this matrix operation is possible in principle, but it is unnecessary for obtaining the flag.

## 5. The Easier Route

Rather than concentrating entirely on the failure/validation path, inspect what the program does when verification succeeds.

The success-side code reconstructs output data using values already present in the obfuscated arrays.

The relevant logic effectively performs:

```text
_FCT + _MFI
      |
      v
reconstruct output
      |
      v
display flag
```

This is the useful shortcut.

The challenge does not require us to recover a valid input mathematically if the verifier already contains enough information to reconstruct the expected result.

## 6. Flag Recovery

Following the success branch and rebuilding the generated string gives:

```text
UNI6CTF{SaRe_Jahan#1947_xYz!}
```

The challenge submission format requires the `TRIVARNA` wrapper, producing:

```text
TRIVARNA{SaRe_Jahan#1947_xYz!}
```

The thematic portion also fits the challenge: `SaRe Jahan` evokes the patriotic phrase “Saare Jahan Se Achha,” while `1947` references Indian independence.

## 7. Complete Solve Path

```text
verifier.blob
      |
      v
Base64 decode
      |
      v
XZ extraction
      |
      v
BZip2 extraction
      |
      v
GZip extraction
      |
      v
verifier.js
      |
      v
inspect obfuscated verifier
      |
      v
locate matrix structures
      |
      v
follow success branch
      |
      v
reconstruct stored output
      |
      v
TRIVARNA{SaRe_Jahan#1947_xYz!}
```

## 8. Takeaways

- A suspicious binary blob may simply be a stack of encoded/compressed layers.
- Recover the real source before attempting to reverse the algorithm.
- In an obfuscated verifier, inspect the success path as well as the input-checking path.
- A complicated matrix transformation does not necessarily need to be inverted if the expected output is reconstructed elsewhere.
- Challenge hints can identify the major internal structure without giving away the implementation.

## Final Flag

```text
TRIVARNA{SaRe_Jahan#1947_xYz!}
```

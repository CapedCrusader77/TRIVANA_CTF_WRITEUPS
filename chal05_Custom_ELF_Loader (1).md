# Custom ELF Loader

## Challenge
**File:** `chal05.apk`  
**Difficulty:** Easy  
**Points:** 100

## Solution
`assets/payload.bin` is encrypted and is decrypted by `libstub.so` through `payload_decrypt`.

The key and IV are embedded in `.rodata`:

```text
STUSEC_AES_KEY26
STUSEC_AES_IV_26
```

Both are 16 bytes.

The native code calls an AES block-encryption routine on incrementing counter blocks and XORs the result with the payload, establishing:

```text
AES-128-CTR
```

Decrypting `payload.bin` produces a second, nested ELF shared object.

The custom loader then manually:

1. Maps the ELF.
2. Applies relocations including `R_*_RELATIVE`.
3. Sets memory permissions with `mprotect`.
4. Resolves dynamic symbols.
5. Locates and executes `get_flag()`.

Inside `get_flag()`, a hardcoded 47-byte `ENC` blob is XORed with the repeating key:

```text
STUSEC_ELF_KEY
```

using:

```text
ENC[i] XOR KEY[(i + 7) % 14]
```

The decoded bytes form the flag body.

## Final Flag

```text
TRIVARNA{c37f_cUs70m_3lf_l04d3r_n0_p4ck3r_s16n47ur3_5521}
```

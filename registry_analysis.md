# Registry Analysis

## Solution

The challenge provides `NTUSER.DAT` and `SYSTEM`. The malicious persistence value is:

```text
UserInitMprLogonScript
```

Raw hive inspection located the value around offset `0x1340`. The hive had a dirty/transactional state, so raw binary analysis was useful when standard parsing was unreliable.

An embedded PowerShell `-EncodedCommand` payload was located around the `0x105c`/`0x10b0` region.

The payload decodes as:

```text
Base64 -> UTF-16LE
```

and reveals:

```text
$flag = "flag{r3g1stry_us3r1n1tmpr_p3rs1st3nc3_2026}"
```

## Final Flag

```text
TRIVARNA{r3g1stry_us3r1n1tmpr_p3rs1st3nc3_2026}
```

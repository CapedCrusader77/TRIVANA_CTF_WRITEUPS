# Timeline Reconstruction

## Challenge

Identify a timestomped NTFS file by comparing `$STANDARD_INFORMATION` and `$FILE_NAME` timestamps, then recover its resident data.

## Solution

The supplied file was:

```text
mft_extract.raw
```

It contained 2,000 MFT records, each 1,024 bytes.

A parser was used to inspect the records and compare the `$SI` and `$FN` timestamps.

The suspicious record was:

```text
Record #1024
File: svchost.dll
```

Its timestamps showed a clear mismatch:

```text
$SI → 2021-01-15 10:00:00
$FN → 2026-07-20 14:30:00
```

The old `$SI` timestamp was the timestomped value, while the `$FN` timestamp remained consistent with the later creation time.

The resident `$DATA` contained an encoded value associated with:

```text
SecretKeyXOR37
```

The important detail was that `37` refers to hexadecimal `0x37`.

XOR each byte with:

```text
0x37
```

The decoded payload became:

```text
flag{ntfs_mft_t1m3st0mp_fn_vs_s1_d1ff}
```

## Flag

```text
TRIVARNA{ntfs_mft_t1m3st0mp_fn_vs_s1_d1ff}
```

## Key Takeaway

NTFS stores multiple timestamp sets. Changing `$STANDARD_INFORMATION` while leaving `$FILE_NAME` timestamps unchanged creates a detectable mismatch that can reveal timestomping.

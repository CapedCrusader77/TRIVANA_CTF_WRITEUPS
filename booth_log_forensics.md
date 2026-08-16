# Booth Log Forensics — Proof of Concept

**Flag:** `UNI6CTF{SaRe@Prabhat_4uM!63}`

## Overview

`booth.log` is another extension trap. Although it looks like an ordinary log, its contents form a ZIP archive. The archive contains three JSON-lines logs, each using a different encoding for its payload.

```text
booth_alpha.log → hexadecimal
booth_beta.log  → Base64
booth_gamma.log → Base32
```

The final flag is reconstructed by filtering the records, ordering them by their real timestamps, and undoing a small transposition.

## 1. Extract the Actual Logs

Treat the file according to its archive signature rather than its `.log` suffix.

After extraction, the three JSONL files can be processed independently. Every record provides enough metadata to determine whether it belongs to the intended ceremony sequence.

## 2. Eliminate Decoy Flags

Several payloads deliberately decode to flag-looking strings:

```text
UNI6CTF{Visible_Sabha_Decoy}
UNI6CTF{Wrong_Date_Signal}
UNI6CTF{Invalid_Relay_Flag}
```

Their surrounding metadata makes them invalid:

```text
PRACTICE instead of CEREMONY
16-08-1947 instead of 15-08-1947
valid = false
```

One invalid alpha record provides the explicit clue:

```text
sort-by-string-is-wrong
```

This points to the timestamp handling issue.

## 3. Sort by Actual Time

The log entries use different timezone offsets. Therefore, lexicographic ordering of the timestamp strings is not equivalent to chronological ordering.

Convert every timestamp to a common timezone first:

```python
from datetime import datetime, timezone

ordered = sorted(
    (
        datetime.fromisoformat(ts).astimezone(timezone.utc),
        name,
        payload
    )
    for name, ts, payload in entries
)
```

The correct candidates are the seven records satisfying:

```text
event/tag = CEREMONY
valid     = true
date      = 15-08-1947
```

## 4. Decode the Seven Fragments

Once decoded and chronologically ordered, the fragments are:

```text
UICF
SR@r
ba_u
!3N6
T{ae
Paht
4M6}
```

Joining them gives:

```text
UICFSR@rba_u!3N6T{aePaht4M6}
```

This is still scrambled, so it is not the final output.

## 5. Undo the Transposition

The 28 characters can be split into two rows of fourteen:

```text
UICFSR@rba_u!3
N6T{aePaht4M6}
```

Read the two rows vertically, taking one character from each row at a time:

```python
concat = "".join(p for _, _, p in ordered)

row0 = concat[:14]
row1 = concat[14:]

flag = "".join(
    first + second
    for first, second in zip(row0, row1)
)

print(flag)
```

The reconstructed text is:

```text
UNI6CTF{SaRe@Prabhat_4uM!63}
```

## 6. Solve Flow

```text
booth.log
   ↓
recognize ZIP container
   ↓
extract alpha / beta / gamma logs
   ↓
decode hex / Base64 / Base32
   ↓
discard explicit decoys
   ↓
keep valid CEREMONY records for 15-08-1947
   ↓
convert timestamps to a common timezone
   ↓
sort by chronological instant
   ↓
join the seven 4-byte fragments
   ↓
undo the 2 × 14 transposition
   ↓
recover final flag
```

## Result

```text
UNI6CTF{SaRe@Prabhat_4uM!63}
```

## Key Point

The challenge combines several small forensic observations: filenames cannot be trusted, decoded flag-shaped values still need validation, timestamps must be compared as timezone-aware instants, and correctly ordered fragments may still require a final transposition step.

# U - Signal Sabha

**Category:** Misc
**Difficulty:** Medium
**Points:** 200

---

## Challenge Description

The Independence Day signal desk collected relay logs from several volunteer
booths before the dawn ceremony. The logs contain real tricolor fragments,
wrong-date rehearsals, fake flags, and timestamps from different time zones.
The final message was split, encoded in mixed bases, and logged out of order.

## Provided Files

```
attachments/
└── booth.log      # actually a ZIP archive (misnamed extension)
```

Unzipping `booth.log` reveals three separate booth export logs:

```
booth_alpha.log
booth_beta.log
booth_gamma.log
```

Each file contains newline-delimited JSON records:

```json
{"timestamp":"1947-08-15T00:42:00+05:30","date":"15-08-1947","tag":"CEREMONY",
 "booth":"alpha","color":"saffron","payload":"344d367d","valid":true}
```

## Step 1 — Unwrap the disguised archive

```bash
file booth.log
# booth.log: Zip archive data, at least v2.0 to extract
cp booth.log booth.zip && unzip booth.zip
```

## Step 2 — Understand the record schema

Fields per record:

- `timestamp` — ISO-8601 with a timezone offset (varies per record — **this is
  one of the confounds mentioned in the brief**)
- `date` — a DD-MM-YYYY string, meant to describe the ceremony date
- `tag` — `CEREMONY` (real ceremony traffic) or `PRACTICE` (rehearsal noise)
- `booth` — `alpha` / `beta` / `gamma`
- `color` — the tricolor band the booth is *supposed* to represent:
  `alpha → saffron`, `beta → white`, `gamma → green`
- `payload` — encoded fragment (hex / base64 / base32 depending on booth)
- `valid` — a boolean sanity flag

## Step 3 — Filter out every decoy

Four records fail one or more integrity checks and are discarded:

| Record | Reason for exclusion |
|---|---|
| `alpha` @ `00:21+00:00` | `valid:false`. Decoding its hex payload gives the plaintext **`sort-by-string-is-wrong`** — a deliberate hint that timestamps must be normalized to a common timezone and compared numerically, not sorted as raw strings. |
| `alpha` @ `00:02+05:30` | `tag:PRACTICE`, not a ceremony transmission. Its base64 payload decodes to `UNI6CTF{Visible_Sabha_Decoy}` — an obvious fake flag. |
| `beta` @ `00:10, date=16-08-1947` | Wrong date **and** its `color` field says `saffron` even though it's booth `beta` (which should always report `white`) — a double red flag. Decodes to `UNI6CTF{Wrong_Date_Signal}`. |
| `gamma` @ `00:18+05:30` | `valid:false`. Payload is a malformed/garbage base32 string. |

The remaining 7 records all satisfy: `tag == CEREMONY`, `valid == true`,
`color` matches the booth's canonical color, and — critically — resolve to
**15-08-1947 in IST** once the timezone offset is applied.

## Step 4 — Normalize timestamps to IST and order

```python
IST = datetime.timezone(datetime.timedelta(hours=5, minutes=30))
ist_time = datetime.datetime.fromisoformat(record['timestamp']).astimezone(IST)
```

Sorting the 7 surviving records by true IST instant reveals a clean,
deliberate cadence: each booth repeats every **21 minutes**, and the three
booths are staggered **8 minutes** apart from each other
(`alpha` @ :00, `beta` @ :08, `gamma` @ :16, then the cycle repeats):

| IST time | Booth | Payload format |
|---|---|---|
| 00:00 | alpha | hex |
| 00:08 | beta | base64 |
| 00:16 | gamma | base32 |
| 00:21 | alpha | hex |
| 00:29 | beta | base64 |
| 00:37 | gamma | base32 |
| 00:42 | alpha | hex |

This tight, evenly-spaced pattern is the confirmation signal that these are
the genuine relay transmissions (the discarded decoys don't fit the cadence).

## Step 5 — Decode each fragment with its booth's base

```python
import base64

def dec(kind, s):
    if kind == "hex":
        return bytes.fromhex(s)
    if kind == "b64":
        return base64.b64decode(s)
    if kind == "b32":
        s += "=" * ((8 - len(s) % 8) % 8)
        return base64.b32decode(s)
```

| Time | Payload | Decoded |
|---|---|---|
| 00:00 | `55494346` (hex) | `UICF` |
| 00:08 | `U1JAcg==` (b64) | `SR@r` |
| 00:16 | `MJQV65I` (b32) | `ba_u` |
| 00:21 | `21334e36` (hex) | `!3N6` |
| 00:29 | `VHthZQ==` (b64) | `T{ae` |
| 00:37 | `KBQWQ5A` (b32) | `Paht` |
| 00:42 | `344d367d` (hex) | `4M6}` |

## Step 6 — Reassemble in chronological order

```python
message = "UICF" + "SR@r" + "ba_u" + "!3N6" + "T{ae" + "Paht" + "4M6}"
# => "UICFSR@rba_u!3N6T{aePaht4M6}"
```

## Flag

```
TRIVARNA{UICFSR@rba_u!3N6T{aePaht4M6}}
```

## Key Takeaways

- Misnamed file extensions are a common misdirection — always `file` a
  suspicious blob before trusting its extension.
- Timezone offsets must be normalized to a single reference frame before
  any chronological ordering/sorting is meaningful; comparing raw ISO
  strings lexicographically silently produces the wrong order.
- Multiple independent integrity signals (validity flag, tag, date,
  color-to-booth consistency) were layered on top of each other — a decoy
  might pass one check but fail another, so every field needs to be
  cross-validated.
- The 8-minute/21-minute relay cadence served as an implicit checksum:
  genuine fragments fit a consistent temporal pattern that decoys did not.

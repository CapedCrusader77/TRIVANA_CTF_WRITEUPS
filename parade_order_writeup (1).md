# Parade Order — Writeup

**Category:** Misc / Forensics
**Difficulty:** Easy
**Points:** 100
**Flag:** `TRIVARNA{Parade_Clues_March_In_Order}`

---

## Challenge Description

> The Independence Day parade office recovered a shuffled set of march cards from
> the midnight rehearsal. Some cards are authentic, some were copied on the wrong
> day, and a few contain tempting flag-like distractions. The real cards were
> stamped on the historic date.

Provided file: `attachments.json`, downloaded from:
```
https://csem.ip-167-235-30-42.swiftwave.xyz/media/event_challenge_files/attachments.json
```

---

## Step 1 — Peeling the Archive Layers

The provided file was not plain JSON despite its `.json` extension — it was a
polyglot chain of nested archive formats, each wrapping the next:

```
attachments.json          → ZIP
 └─ attachments/parade_cards.json   → GZIP
     └─ (decompressed)              → TAR
         └─ parade_cards.json       → ZIP (again)
             └─ parade_cards.json   → plain JSON  ✅
```

Unwrapped with a short Python loop that auto-detected magic bytes (`PK\x03\x04`
for ZIP, `\x1f\x8b` for GZIP) and recursed until valid JSON was reached:

```python
import zipfile, gzip, io, json

def detect(data):
    if data[:2] == b'PK': return 'zip'
    if data[:2] == b'\x1f\x8b': return 'gzip'
    try:
        json.loads(data); return 'json'
    except Exception:
        return 'unknown'

data = open('attachments.json', 'rb').read()
while True:
    t = detect(data)
    if t == 'zip':
        zf = zipfile.ZipFile(io.BytesIO(data))
        data = zf.read(zf.namelist()[-1])   # inner file
    elif t == 'gzip':
        data = gzip.decompress(data)
    elif t == 'json':
        break
```

(The intermediate GZIP layer actually decompressed to a **tar** stream, so `tar
-xzf` was used directly at that stage instead of a raw gzip read.)

The final payload was `parade_cards.json`:

```json
{
  "title": "Parade Order",
  "theme": "Independence Day",
  "note": "The real parade cards were stamped on 15-08-1947.",
  "cards": [
    {
      "march": 22,
      "color": "white",
      "pick": 17,
      "phrase": "ZyUd5Zmuib3yP2_SMpeBMmrame!O",
      "stamp": "d5b41d2cbb"
    },
    ...
  ]
}
```

71 cards total, each with:
- `march` — a claimed position number in the parade
- `color` — one of `saffron`, `white`, `green`, `chakra`
- `pick` — an index into `phrase`
- `phrase` — a noisy string of characters
- `stamp` — a 10-hex-char checksum

Two cards were obvious **decoys** — full flag-shaped strings sitting directly
in the `phrase` field:

```
"phrase": "UNI6CTF{Wrong_March_Wrong_Message}"
"phrase": "UNI6CTF{Tricolor_Decoy_1947}"
```

Grabbing either of these directly is the trap the challenge description warns
about ("a few contain tempting flag-like distractions").

---

## Step 2 — Reverse-Engineering the Stamp

The note *"The real parade cards were stamped on 15-08-1947"* pointed at the
`stamp` field being some kind of date-derived checksum. A brute-force search
over common date formats, field orderings, separators, and hash algorithms
against one known card recovered the formula:

```python
stamp = sha256(f"{date}|{march}|{color}|{pick}")[:10]     # date = "15-08-1947"
```

Filtering all 71 cards against this formula isolated exactly **36 authentic
cards** — and their `march` values turned out to be a clean, unique
run from **1 to 36**, i.e. the real parade order. The two decoy cards
(and everything else) failed the stamp check and were discarded.

```python
date = "15-08-1947"
authentic = [
    c for c in cards
    if sha256(f"{date}|{c['march']}|{c['color']}|{c['pick']}".encode())
         .hexdigest()[:10] == c['stamp']
]
authentic.sort(key=lambda c: c['march'])   # 1 → 36
```

---

## Step 3 — Extracting the Flag

Sorting the 36 authentic cards by `march` and pulling the character at index
`pick - 1` (1-indexed) from each card's `phrase` string spelled out the flag,
letter by letter, in parade order:

```python
flag_body = ''.join(c['phrase'][c['pick'] - 1] for c in authentic)
```

Output:

```
Parade_Clues_March_In_Order
```

Wrapped in the event's flag format:

```
TRIVARNA{Parade_Clues_March_In_Order}
```

---

## Final Flag

```
TRIVARNA{Parade_Clues_March_In_Order}
```

## Key Takeaways

- **Don't trust file extensions** — `.json` was actually a 4-layer nested
  archive (ZIP → GZIP → TAR → ZIP → JSON).
- **Flag-shaped strings aren't always the flag** — two decoy cards embedded
  full `UNI6CTF{...}`-style strings directly to bait a quick submit.
- **The narrative is the algorithm** — the "stamped on the historic date"
  hint directly encoded the hash input used to validate authentic cards.
- **Ordering matters** — the `march` field wasn't just metadata; it was the
  index needed to reassemble the flag in the correct sequence.

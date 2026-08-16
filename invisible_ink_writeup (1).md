# Invisible Ink — Writeup

**Category:** Misc / Forensics
**Points:** 200
**Files:** `compliance_report.pdf` (built with `reportlab` + `pikepdf`)

## TL;DR

`TRIVARNA{0ffs3t_0rd3r_n0t_0bj3ct_0rd3r}`

## Recon

Opening the PDF just shows a boring "Device Compliance Report" with a status
line. Extracting text (or copy-pasting from a viewer) gives:

```
Device Compliance Report
Status: OK. Nothing else to see here.
CSEMA{wh1t3_t3xt_1s_n07_1t}
```

That flag-shaped string is the bait.

## Step 1 — Inspect the content stream

Pulling the raw page content stream with `pikepdf`:

```python
import pikepdf
pdf = pikepdf.open('compliance_report.pdf')
print(pdf.pages[0].Contents.read_bytes())
```

```
... 1 1 1 rg
BT /F1 10 Tf 12 TL ET
BT 1 0 0 1 72 680 Tm (CSEMA{wh1t3_t3xt_1s_n07_1t}) Tj T* ET
0 0 0 rg
```

`1 1 1 rg` sets the fill color to white immediately before that string is
drawn — literal white-on-white text. The string itself even spells it out:
`wh1t3_t3xt_1s_n07_1t` → "white text is not it". Confirmed decoy.

## Step 2 — Look past the page contents

The real payload isn't in the visible/text layer at all — it's stashed in
the page's `/Resources` dictionary, under a custom, non-standard key
(`/CSEMFragments`) that PDF viewers ignore because it isn't a resource type
they know how to render:

```python
pdf.pages[0]['/Resources']['/CSEMFragments']
```

This is an array of 6 small dictionaries, each with a decoy `/Note` field
("device audit fragment") and one differently-named key holding a fragment
of the real flag:

| Key    | Value      |
|--------|------------|
| /SegC  | `CSEMA{`   |
| /SegF  | `0ffs3t`   |
| /SegA  | `_0rd3r`   |
| /SegD  | `_n0t_0`   |
| /SegB  | `bj3ct_`   |
| /SegE  | `0rd3r}`   |

The key names (`SegA`..`SegF`) are deliberately shuffled/alphabetically
out of order — a trap for anyone tempted to just sort by key name.

## Step 3 — Determine the real ordering

The flag's own content is a hint about how to order the fragments:
**"offset order, not object order."** To confirm which ordering is
authoritative, three independent checks were run — all agreed:

1. **Array order** as stored in `/CSEMFragments`: C, F, A, D, B, E
2. **PDF indirect object numbers** (`objgen`) of each fragment dict:
   `6, 7, 8, 9, 10, 11` — same sequence as the array
3. **Actual byte offsets** in the raw file, cross-checked two ways
   (manual regex scan for `N 0 obj`, and `pypdf`'s parsed xref/trailer
   table): `897, 963, 1029, 1095, 1161, 1228` — same sequence again

```python
import re
data = open('compliance_report.pdf','rb').read()
for m in re.finditer(rb'(\d+) 0 obj', data):
    print(int(m.group(1)), m.start())
```

All three orderings coincide, so the fragments concatenate cleanly:

```
CSEMA{ + 0ffs3t + _0rd3r + _n0t_0 + bj3ct_ + 0rd3r}
= CSEMA{0ffs3t_0rd3r_n0t_0bj3ct_0rd3r}
```

## Flag

Challenge flag format for this event is `TRIVARNA{...}`, so swap the
prefix:

```
TRIVARNA{0ffs3t_0rd3r_n0t_0bj3ct_0rd3r}
```

## Takeaways

- Don't trust the first flag-shaped string you see — check fill color /
  render state around any text draw operator (`rg`, `RG`, `Tr` render
  mode 3, etc.) for invisible-text tricks.
- Non-standard keys inside `/Resources` (or anywhere in the object graph)
  are a common place to hide CTF payloads, since viewers silently ignore
  keys they don't recognize.
- When fragment ordering is ambiguous, cross-check multiple independent
  signals (array order, object numbers, physical byte offsets via the
  xref table) rather than trusting just one.

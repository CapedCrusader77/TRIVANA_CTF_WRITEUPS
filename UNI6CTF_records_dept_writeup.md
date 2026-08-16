# UNI6CTF — "Records Department" Forensics Challenge Writeup

**Category:** Forensics
**Challenge files:** `candidate_passwords.txt`, `UNI6CTF_Quarterly_Report.pdf`
**Flag:** `UNI6CTF{m3t4d4t4_wh1sp3rs_wh1l3_p1x3ls_h1d3}`

---

## Summary

The challenge ships an encrypted PDF disguised as a mundane quarterly finance
report. Solving it requires chaining four separate hiding techniques stacked
inside the same document: password-protected PDF → metadata encoding →
LSB steganography in an embedded chart image → steghide-in-BMP with a
passphrase recovered from that LSB payload → XOR keyed off the metadata
value. One of the two "clue" strings found early on (a Base91-encoded PDF
annotation) turns out to be a deliberate red herring.

---

## 1. Initial recon

Unzipping the challenge archive gives exactly two files:

```
candidate_passwords.txt
UNI6CTF_Quarterly_Report.pdf
```

`candidate_passwords.txt` lists several plausible PDF passwords plus a
placeholder comment:

```
NightShift_2025
QuietLedger_2025
GhostProtocol_2025
Knightshift_2024
Nightshift_2025
# INSERT_YOUR_3_LAYER_ENCODED_ENTRY_HERE
```

None of the listed candidates work verbatim. The correct password —
found by testing capitalization/punctuation variants — is:

```
Knightshift_2025
```

(Capital K, no trailing `!`.)

## 2. Decrypting the PDF

```bash
qpdf --password='Knightshift_2025' --decrypt UNI6CTF_Quarterly_Report.pdf decrypted.pdf
```

This produces a normal-looking 2-page finance report ("Quarterly
Performance Report, FY2025") with an executive summary, a revenue chart,
and a departmental budget table. `pdftotext -layout` confirms there's
nothing hidden in the visible body text.

## 3. First clue — PDF metadata

```bash
exiftool decrypted.pdf
```

```
Keywords : NGIzZTkxZDI3NzBh
Subject  : Internal financial report - see records dept for archival ID.
```

Base64-decoding the `Keywords` field:

```bash
echo 'NGIzZTkxZDI3NzBh' | base64 -d
# → 4b3e91d2770a
```

Call this **A = 4b3e91d2770a**.

## 4. Second clue — PDF annotation

Unpacking the PDF's object structure to inspect annotations:

```bash
qpdf --qdf --object-streams=disable decrypted.pdf unpacked.pdf
grep -A5 "/Subtype /Text" unpacked.pdf
```

```
/Contents (archival ref \(legacy, pre-2024 format\): F%QGR]aJo"[k6{?)
```

The string `F%QGR]aJo"[k6{?` decodes cleanly with the standard **basE91**
alphabet:

```
205746dcf4a66102763466ff
```

Call this **B**. At this point it's tempting to assume A and B need to be
combined (XOR, RC4, concatenation, etc.) to yield the flag — **this is a
dead end**. Exhaustively testing every reasonable A/B combination produces
no meaningful plaintext, because B is not part of the real key chain at
all; it's a distractor planted alongside a legitimately-decodable (and
therefore convincing) encoding.

## 5. The real next step — LSB steganography in the embedded image

The PDF's "Figure 1: Quarterly Revenue vs. Operating Costs" chart is an
embedded raster image. Extracting it:

```bash
pdfimages -all decrypted.pdf images/img
```

gives a 900×600 RGB PNG. Standard automated stego scanners (`zsteg`,
`pngcheck`, `exiftool`) report nothing interesting — because the payload
sits specifically in the **blue channel's least-significant bits**, which
generic scans don't always surface cleanly depending on bit-plane/channel
ordering. Extracting that plane manually:

```python
from PIL import Image
import numpy as np

img = Image.open('img-000.png').convert('RGB')
flat = np.array(img).reshape(-1, 3)
bits = flat[:, 2] & 1          # blue channel LSBs
data = np.packbits(bits[:len(bits) - len(bits) % 8]).tobytes()
```

Searching the recovered byte stream for printable text reveals a
null-terminated message buried in a sea of `0xFF` padding:

```
CHECK STEGHIDE PASSPHRASE: nightshift
```

This is the actual pivot the challenge wants: the PDF metadata/annotation
strings are decorative "archival ID" flavor text, but the *real* records
dept clue is hidden inside the pixels of the chart itself.

## 6. steghide extraction

`steghide` doesn't support PNG as a cover format (only JPEG/BMP/WAV/AU),
so the PNG is losslessly re-saved as BMP first:

```python
Image.open('img-000.png').save('img-000.bmp')
```

Then extracted with the recovered passphrase:

```bash
steghide extract -sf img-000.bmp -p "nightshift" -xf out_secret.txt
```

This succeeds and yields 44 bytes of binary data — still not
human-readable, so one more transform is needed.

## 7. Final decode — XOR with the metadata value

XORing the extracted 44 bytes with a repeating-key of **A**
(`4b3e91d2770a`, the value recovered from the PDF `Keywords` field back in
step 3) produces 100% printable ASCII:

```python
def xor_repeat(data, key):
    return bytes(d ^ key[i % len(key)] for i, d in enumerate(data))

xor_repeat(open('out_secret.txt','rb').read(), bytes.fromhex('4b3e91d2770a'))
# → b'UNI6CTF{m3t4d4t4_wh1sp3rs_wh1l3_p1x3ls_h1d3}'
```

## Flag

```
UNI6CTF{m3t4d4t4_wh1sp3rs_wh1l3_p1x3ls_h1d3}
```

Fittingly, the flag text itself describes the solve path: metadata
("whispers") pointed toward the pixels ("hide"), which is where the real
steganographic chain actually lived.

---

## Full solve chain (diagram)

```
ZIP
 └─ UNI6CTF_Quarterly_Report.pdf   [password: Knightshift_2025]
     ├─ Metadata Keywords (Base64) ─────────────► A = 4b3e91d2770a
     ├─ Annotation (Base91)  ───► B = 205746dcf4a66102763466ff   [RED HERRING]
     └─ Embedded chart PNG
          └─ Blue-channel LSB  ─────────────────► "steghide passphrase: nightshift"
               └─ PNG → BMP → steghide extract (pw: nightshift)
                    └─ 44-byte ciphertext
                         └─ XOR with A ──────────► FLAG
```

## Key lessons for future challenges

- **Don't assume every recovered "clue" is meant to combine with every
  other clue.** B was real, decodable, and thematically consistent
  ("archival ref, legacy format") — but it was never meant to be used. A
  clue that decodes cleanly isn't proof it's load-bearing.
- **When automated stego tools (`zsteg`) come back empty, check
  channels/bit-planes manually.** Default tool heuristics don't always
  hit the exact channel a challenge author picked.
- **File-format mismatches are sometimes the puzzle.** `steghide` requires
  BMP/JPEG/WAV/AU; a lossless PNG→BMP conversion was required before the
  tool would even attempt extraction.
- **Layered stego (image LSB → steghide → XOR) is a common CTF pattern**:
  one layer's output is the *key or passphrase* for the next layer, not
  the flag itself.

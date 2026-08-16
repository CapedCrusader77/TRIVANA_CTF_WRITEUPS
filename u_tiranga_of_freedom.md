# U — Tiranga of Freedom

**Challenge:** UNI6CTF — Independence Day Terminal Capture  
**Category:** Misc  
**Flag:** `UNI6CTF{r3sizing_7h3_indi4n_fl4g}`

## 1. Challenge Overview

The supplied archive contains a text file named `Challenge misc.txt`. At first glance, it appears to be a corrupted terminal dump: long rows of punctuation, ASCII-art fragments, and several injected status messages.

The key clue is that the terminal output was supposedly captured using an incorrect line width.

This means the visible lines should not be treated as genuine rows of the artwork. They are pieces of one longer character sequence that has been wrapped incorrectly.

The solve therefore consists of:

1. Removing the injected status messages.
2. Reconstructing the uninterrupted character stream.
3. Recovering the original display width.
4. Rebuilding the two-dimensional artwork.
5. Interpreting the resulting banner.

---

## 2. Inspecting the Capture

The file contains 68 CRLF-separated lines. Several begin with comments such as:

```text
# redraw jitter NN: ignore this terminal status line
```

These are not part of the artwork.

The remaining lines should also **not** be joined with spaces or newline characters. Their line breaks are artifacts introduced by the incorrect terminal wrapping.

A minimal reconstruction is:

```python
with open("Challenge misc.txt", "r", encoding="utf-8", newline="") as f:
    raw = f.read()

parts = raw.split("\r\n")
stream = "".join(line for line in parts if not line.startswith("#"))

print(len(stream))
```

The recovered one-dimensional stream contains:

```text
5760
```

characters.

---

## 3. Finding the Original Width

The next problem is determining how many characters belonged on each original terminal row.

A useful clue survives inside the flattened stream: repeated `>>` and `<<` sequences belonging to diagonal elements of the ASCII artwork.

Search for their absolute positions:

```python
import re

gt_positions = [m.start() for m in re.finditer(r">>", stream)]
lt_positions = [m.start() for m in re.finditer(r"<<", stream)]

print(gt_positions)
print(lt_positions)
```

The `>>` occurrences appear at:

```text
902, 1030, 1158, 1286, 1414
```

Each consecutive occurrence is separated by:

```text
128
```

The second marker family independently gives:

```text
4982, 5110, 5238, 5366, 5494
```

which has the same spacing.

So the original row width is:

```text
128 columns
```

There is an additional consistency check:

```text
5760 / 128 = 45
```

with no remainder.

Therefore the original artwork is exactly:

```text
45 rows × 128 columns
```

---

## 4. Reconstructing the Artwork

Split the recovered stream into 128-character rows:

```python
width = 128

grid = [
    stream[i:i + width]
    for i in range(0, len(stream), width)
]
```

The result is a clean 45-row terminal grid.

Before reconstruction, the punctuation looks meaningless. After restoring the correct width, its structure becomes visible.

The upper and lower portions form ASCII lettering, while the central section resembles the Indian national flag.

The broad layout is:

```text
Rows  1–13   → banner artwork
Rows 14–37   → Tiranga / flag artwork
Rows 38–45   → banner artwork
```

The center contains the three horizontal color regions represented through different punctuation characters, with an ASCII approximation of the Ashoka Chakra in the white band.

---

## 5. Investigating Unexpected Characters

Most of the reconstructed grid uses characters belonging to the artwork's expected palette.

A scan for unexpected characters reveals a particularly interesting cluster:

```text
row 15:  U   n
row 16:  N   d
row 17:  I   i
row 18:  6   4
```

Reading vertically gives:

```text
UNI6
ndi4
```

The first column clearly produces:

```text
U
N
I
6
```

This is significant because it corresponds to the competition identifier.

The characters are not random corruption. They are remnants of the rendering process: parts of the intended glyph labels leaked into the terminal artwork when particular characters failed to render normally.

The other punctuation-like characters around the Ashoka Chakra are consistent with the artwork and should not be treated as independent flag data.

---

## 6. Rendering the Recovered Grid

The banner is graphical rather than ordinary plaintext, so rendering the reconstructed grid makes the hidden lettering much easier to inspect.

For example:

```python
from PIL import Image

scale = 10
height = len(grid)
width = len(grid[0])

image = Image.new(
    "RGB",
    (width * scale, height * scale),
    "black"
)

pixels = image.load()

for r, row in enumerate(grid):
    for c, char in enumerate(row):
        visible = char in "/\\v<>"

        value = (255, 255, 255) if visible else (20, 20, 20)

        for dy in range(scale):
            for dx in range(scale):
                pixels[
                    c * scale + dx,
                    r * scale + dy
                ] = value

image.save("banner.png")
```

The resulting image exposes the banner text much more clearly than the original wrapped capture.

---

## 7. Extracted Banner

The reconstructed lettering spans multiple lines because the original artwork is wider than a convenient text display.

It reads:

```text
UNI6CTF{
r3sizing_7h3
indi4n_fl4g}
```

Combining the lines gives:

```text
UNI6CTF{r3sizing_7h3_indi4n_fl4g}
```

The leetspeak wording describes the central trick:

```text
r3sizing 7h3 indi4n fl4g
```

or, normally written:

```text
resizing the indian flag
```

That directly reflects the required operation: restoring the incorrectly wrapped terminal output to its original dimensions.

---

## 8. Reproducible Solve

The complete workflow can be summarized as:

```text
Challenge misc.txt
        │
        ▼
Remove # status lines
        │
        ▼
Concatenate remaining lines
        │
        ▼
5760-character stream
        │
        ▼
Detect repeated marker spacing
        │
        ▼
128-character width
        │
        ▼
45 × 128 reconstructed grid
        │
        ▼
Render ASCII artwork
        │
        ▼
Read banner text
        │
        ▼
UNI6CTF{r3sizing_7h3_indi4n_fl4g}
```

An end-to-end solver should report approximately:

```text
[*] 68 raw lines total
[*] 6 injected/comment lines removed
[*] Clean stream length: 5760 characters
[*] Marker spacing confirms width = 128
[*] Reconstructed grid = 45 rows × 128 columns
[*] Banner rendered successfully
[*] Flag recovered
```

The reconstruction, width detection, and rendering can all be automated. The final interpretation of the stylized banner can be done manually or with OCR.

---

## 9. Final Flag

```text
UNI6CTF{r3sizing_7h3_indi4n_fl4g}
```

## 10. Key Lessons

- Do not automatically trust line breaks in terminal captures.
- Remove explicitly injected status/comment lines before reconstruction.
- When ASCII art has been rewrapped, flattening the data first can reveal periodic structures.
- Repeated marker offsets are useful for recovering an unknown row width.
- An exact division of the stream length by the candidate width is a strong sanity check.
- Once the correct geometry is restored, apparently random punctuation can resolve into meaningful artwork.

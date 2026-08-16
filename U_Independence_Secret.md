# U --- Independence Secret: The Hidden Image Trail

**Platform:** UNI6CTF / Trivarna\
**Category:** Steganography / Forensics\
**Difficulty:** Hard

## 1. Challenge objective

The challenge is built as a multi-stage image investigation. Instead of
placing the complete flag in one location, it distributes the answer
across several hidden-data layers.

The solve requires following an image/QR chain, extracting information
from individual color channels, decoding several text transformations,
and finally combining the recovered fragments.

The important rule is to treat each discovery as either:

-   a navigation clue that points to the next artifact, or
-   an actual piece of the final flag.

The two should not be confused.

## 2. Initial observations

The first image contains a QR-driven path toward another image.

The major techniques encountered during the solve are:

``` text
QR / image inspection
        ↓
low-contrast text
        ↓
blue-channel LSB
        ↓
Base62
        ↓
steghide
        ↓
ROT8000
        ↓
URL decoding
        ↓
PNG LSB
        ↓
ROT47
```

There is also repeated flag-like text in metadata. That content is a
deliberate distraction and should not be accepted as the final answer
merely because it looks convincing.

## 3. Recover the first fragment

Begin with `image1.png`.

The first useful hidden element is not immediately obvious at normal
brightness. Increasing the contrast of the lower-right portion exposes
faint text.

The recovered beginning is:

``` text
UNI6CTF{Y1s
```

This is only the first fragment. The closing brace is absent, which is a
useful indication that more data still has to be recovered.

## 4. Inspect the image channels

The next clue is hidden at the bit level.

Extract the least significant bit stream from the blue channel of the
image. The resulting data resolves to:

``` text
https://bit.ly/trivarna2026
```

Following that address continues the challenge chain and provides the
next image/artifact.

This is an important distinction: the URL is a **navigation payload**,
not another flag fragment.

## 5. Decode the long encoded stage

The next stage contains a long encoded string.

Treating it as Base62 produces a Google Drive URL pointing toward:

``` text
image4.bin
```

The extension is intentionally unhelpful, so the file should be examined
according to its actual contents rather than its filename.

## 6. Extract the hidden data from image4.bin

The artifact can be treated as a JPEG for the next steganographic
operation.

Using `steghide`:

``` bash
steghide extract -sf image4.bin -p Unrecognized -xf payload.txt
```

produces the hidden payload.

The extracted material contains two useful pieces of information:

1.  another flag fragment, encoded with ROT8000;
2.  a doubly URL-encoded reference leading to the final PNG.

The ROT8000 layer therefore has to be decoded before the fragment
becomes readable.

## 7. Follow the final PNG

After reversing the URL encoding and retrieving the final PNG, inspect
its least significant bit data.

The hidden stream provides another encoded value. Applying ROT47 to that
result reveals the third and final flag fragment.

At this point the three pieces are:

``` text
Part 1
Part 2
Part 3
```

They must be concatenated in order. The individual pieces should not be
independently reformatted or "corrected" before joining them.

## 8. Why the metadata is misleading

One of the traps in the challenge is repeated metadata containing
flag-looking text.

It is tempting to stop when a value resembles:

``` text
UNI6CTF{...}
```

However, the actual solution path continues through the hidden image
data.

The stronger evidence is the consistency of the chain:

``` text
image → URL → encoded artifact → steghide payload → final PNG → final fragment
```

Only after completing that chain do the fragments form the expected
final flag.

## 9. Complete attack path

``` text
image1.png
   |
   +--> enhance lower-right
   |       |
   |       +--> UNI6CTF{Y1s
   |
   +--> blue-channel LSB
           |
           +--> https://bit.ly/trivarna2026
                    |
                    v
              next image/data
                    |
                    v
                 Base62
                    |
                    v
            Google Drive URL
                    |
                    v
               image4.bin
                    |
                    v
                steghide
                    |
                    v
              payload.txt
              /         \
             /           \
       ROT8000        encoded URL
          |                |
          v                v
       Part 2          final PNG
                           |
                           v
                       PNG LSB
                           |
                           v
                         ROT47
                           |
                           v
                        Part 3

Part 1 + Part 2 + Part 3
             |
             v
UNI6CTF{Y1s_1nd4pe1enc6_S4cre1_0ut}
```

## 10. Key techniques

### Low-contrast image recovery

Increasing local contrast can reveal text that is visually present but
nearly indistinguishable from the surrounding image.

### Channel-specific LSB extraction

A hidden message may exist only in one RGB channel. Checking the blue
channel's least significant bits was enough to recover the next URL.

### Multiple encoding layers

The challenge deliberately mixes several transformations. Recognizing
the expected output at each stage is important because a decoded URL,
compressed-looking value, or readable phrase can indicate which
operation should be attempted next.

### Steganographic extraction

The `steghide` stage demonstrates why a file's extension should not be
treated as authoritative. The contents and applicable file format matter
more than the name.

### Decoy recognition

Flag-shaped metadata is not automatically the answer. The challenge
requires completing the hidden-data chain and validating that all
fragments combine into a coherent final flag.

## 11. Practical lessons

-   Preserve the original lossless image whenever possible.
-   Inspect individual RGB channels rather than relying only on the
    visible image.
-   Keep navigation values separate from actual flag fragments.
-   Apply decoding transformations in the order suggested by the
    previous layer.
-   Do not normalize or alter unusual spelling and leetspeak when
    assembling the final answer.
-   A plausible flag found early can be a decoy; validate it against the
    complete extraction chain.

## Final flag

``` text
UNI6CTF{Y1s_1nd4pe1enc6_S4cre1_0ut}
```

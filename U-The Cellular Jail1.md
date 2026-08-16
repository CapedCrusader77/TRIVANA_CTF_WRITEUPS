# U - The Cellular Jail

**Category:** Classical Crypto / Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Platform:** UNI6CTF / Trivarna

---

## Challenge Overview

The challenge provided two artifacts that initially looked unrelated:

```text
attachments/
├── handbill.png
└── cipher.png
```

The handbill contained ordinary revolutionary text, while the second image contained a strip of numbered pairs such as:

```text
26//1  1//2  1//3 ...
```

The important clue was that the two artifacts had been separated and later reunited.

That strongly suggested that one artifact was the **key/codebook** and the other was the **ciphertext**.

---

## 1. Identify the Cipher

The numbered values were written as pairs:

```text
A//B
```

There were two obvious interpretations:

```text
line // word
```

or:

```text
word // letter
```

The first interpretation didn't fit because the handbill contained only around 15 lines, while some first values were much larger.

The second interpretation made sense:

```text
A//B
    ↓
Letter B of word A
```

So the handbill was being used as a **book cipher**.

---

## 2. Extract the Challenge Files

The supplied archive was:

```text
attachments.tgz
```

Extract it with:

```bash
tar -xzf attachments.tgz
```

This gives:

```text
attachments/handbill.png
attachments/cipher.png
```

The `handbill.png` is the codebook and `cipher.png` contains the coordinate pairs.

---

## 3. Transcribe the Handbill

The handbill had to be transcribed **exactly as printed**.

Two details were especially important:

- `hand bill` appears as two separate words.
- One line contains the truncated text `th`.

These imperfections are significant because the cipher uses a continuous word count.

Numbering the words from 1 gives **141 words**.

Some useful checkpoints were:

```text
Word 23  → voice
Word 101 → cannot
Word 110 → cell
Word 129 → next
```

These checkpoints are useful for confirming that the transcription and indexing are correct.

---

## 4. Read the Cipher Strip

The cipher strip contains 79 coordinate pairs.

A few examples are:

```text
26//1
1//2
1//3
11//1
11//4
15//1
...
```

Each pair means:

```text
word number // letter number
```

The low-resolution image made some of the digits difficult to read, so the rows could be cropped and enlarged before transcription.

For example:

```python
from PIL import Image

im = Image.open('cipher.png').convert('L')

for i in range(8):
    top = 20 + i * 52
    im.crop(
        (5, top, 415, top + 42)
    ).resize(
        (1640, 168),
        Image.LANCZOS
    ).save(f'row{i+1}.png')
```

---

## 5. Decode the Coordinates

Once the handbill has been transcribed, each pair can be converted into a character.

Conceptually:

```text
26//1
  ↓
Word 26
  ↓
Take letter 1
  ↓
Character
```

A simple Python decoder can be used:

```python
text = open('handbill.txt').read()
words = [w.strip('.,').lower() for w in text.split()]

out = ''.join(
    words[a-1][b-1]
    for a, b in pairs
)

print(out)
```

The first decoded result is close to readable English, confirming that the **word//letter** interpretation is correct.

---

## 6. Validate the Indexing

There are some very useful built-in checks.

For example, the cipher contains:

```text
23//1
23//2
23//3
23//4
23//5
```

Word 23 is:

```text
voice
```

Therefore those five coordinates produce:

```text
v o i c e
```

Another checkpoint is word 101:

```text
cannot
```

and the sequence:

```text
101//1
101//2
101//3
101//4
```

produces:

```text
c a n n
```

These repeated runs make it clear that the word numbering is correct.

---

## 7. Deal With the Small Decoding Errors

The raw output is:

```text
therecfgnitionphoaseisstopuniiixctfthavoicecanneverbechaineddtopddssroythisnote
```

It is clearly intended to be readable English, but several coordinates contain small off-by-one errors.

The surrounding context allows these to be corrected.

The intended message is:

```text
THE RECOGNITION PHRASE IS STOP <TAG> THE VOICE CAN NEVER BE CHAINED STOP DESTROY THIS NOTE
```

The important payload is:

```text
THE VOICE CAN NEVER BE CHAINED
```

The `STOP` sections behave like telegraph-style separators.

---

## 8. Recover the Flag

The meaningful phrase is:

```text
THE VOICE CAN NEVER BE CHAINED
```

Convert it to the expected flag format:

```text
TRIVARNA{THE_VOICE_CAN_NEVER_BE_CHAINED}
```

---

## Complete Solve Chain

```text
attachments.tgz
       ↓
handbill.png + cipher.png
       ↓
Recognize book-cipher structure
       ↓
Handbill = codebook
       ↓
cipher.png = word//letter coordinates
       ↓
Transcribe handbill exactly
       ↓
Number words continuously
       ↓
Map coordinates to letters
       ↓
Repair small indexing slips using context
       ↓
THE VOICE CAN NEVER BE CHAINED
       ↓
TRIVARNA{THE_VOICE_CAN_NEVER_BE_CHAINED}
```

---

## Flag

```text
TRIVARNA{THE_VOICE_CAN_NEVER_BE_CHAINED}
```

---

## Key Takeaways

- Two apparently unrelated artifacts can form the key and ciphertext of a book cipher.
- The coordinate ranges reveal the correct indexing scheme.
- The handbill must be transcribed exactly; even apparent printing mistakes can affect later word positions.
- Repeated coordinate sequences are useful for validating word numbering.
- When a classical cipher produces almost-readable text, context can help identify small encoding/indexing mistakes.

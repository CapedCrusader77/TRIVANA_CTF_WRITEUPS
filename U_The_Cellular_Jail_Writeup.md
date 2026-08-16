# U — The Cellular Jail — CTF Writeup

## Challenge

**U — The Cellular Jail** — Classical Crypto / Forensics — **200 points**

> During a routine inspection, prison officials recovered two seemingly unrelated items hidden inside the lining of a prisoner's jacket. The first was an ordinary revolutionary handbill encouraging resistance against British rule. The second was a narrow strip of paper covered only with pairs of numbers. The wardens dismissed the numbered note as meaningless and archived both items separately. Decades later, the two pieces resurfaced together in the same evidence folder.

Files provided:

- `attachments.tgz` — a single archive that unpacks to `attachments/`:
  - `handbill.png` — a printed revolutionary handbill (plain English text)
  - `cipher.png` — a narrow strip of paired numbers, e.g. `26//1  1//2  1//3 ...`

Unpack it first:

```bash
tar -xzf attachments.tgz
# → attachments/handbill.png
# → attachments/cipher.png
```

The two artifacts were archived *separately* and only later reunited — that reunion is the whole hint: **one artifact is the key, the other is the ciphertext.**

## Flag

```
TRIVARNA{THE_VOICE_CAN_NEVER_BE_CHAINED}
```

---

## 1. Identify the cipher type

Two files that were deliberately stored apart, then "resurfaced together," is the classic setup for a **book cipher** (also called a running-key or dictionary cipher):

- One file is ordinary readable text — the **key / codebook**.
- The other is a list of coordinates that index *into* that text.

The number strip is formatted as pairs separated by `//`:

```
26//1   1//2   1//3   11//1   11//4 ...
```

Each token is `A//B`. Two natural readings exist for a book cipher:

1. **line // word** — the B-th word on line A, or
2. **word // letter** — the B-th letter of word A.

Because the B-values are small (mostly 1–5) and some A-values are large (up to 129), the pairs cannot be *line//word* (the handbill has only 15 lines). They must be **word // letter**: word number `A` (counted continuously across the whole handbill), letter number `B` of that word.


## 2. Transcribe both artifacts

### 2a. The handbill

Transcribe the handbill exactly as printed — **including its imperfections**, because the word indices depend on them. Two quirks matter:

- Line 9 prints "hand bill" as **two** words (`hand` `bill`), not one.
- Line 14 prints a truncated "**th**" (in "One generation plants th seed…").

Numbering every word continuously (1-indexed) gives **141 words**. A few reference points used later:

| Word # | Word |
|---|---|
| 23 | voice |
| 101 | cannot |
| 110 | cell |
| 129 | next |

### 2b. The cipher strip

Read the strip left-to-right, top-to-bottom. At low resolution several digits are ambiguous, so crop and upscale each row before transcribing. The full sequence of 79 pairs is:

```
26//1  1//2  1//3  11//1  11//4  15//1  3//2  24//1  12//1  4//1
31//2  16//1  26//2  4//2   2//1   5//1   19//2 27//1  42//4  5//4
72//1  72//2 31//1  54//1   33//2  41//1  9//1  9//2   81//1  83//1
63//1  129//3 110//1 60//2  19//1  48//1  54//2 54//3  23//1  23//2
23//3  23//4 23//5  101//1  101//2 101//3 101//4 8//1  107//2 107//1
25//1  97//1 97//2  84//1   92//2  92//3  20//2 51//1  51//4  9//3
82//1  98//1 35//1  13//1   86//1  18//1  18//3 31//1  31//3  106//2
44//4  93//1 93//2  78//3   78//4  62//1  62//2 62//3  94//2
```

> **Technique — reading a low-quality strip.** Upscale each row ~4× with Lanczos resampling so the digits before `//` are legible; the second number is almost always a single small digit. In Python/Pillow:
> ```python
> from PIL import Image
> im = Image.open('cipher.png').convert('L')
> for i in range(8):
>     top = 20 + i*52
>     im.crop((5, top, 415, top+42)).resize((1640, 168), Image.LANCZOS).save(f'row{i+1}.png')
> ```

## 3. Decode

Map each pair `A//B` → letter `B` of word `A` of the handbill.

```python
text = open('handbill.txt').read()               # the transcribed handbill
words = [w.strip('.,').lower() for w in text.split()]

pairs = [(26,1),(1,2),(1,3),(11,1),(11,4),(15,1),(3,2),(24,1),(12,1),(4,1),
         (31,2),(16,1),(26,2),(4,2),(2,1),(5,1),(19,2),(27,1),(42,4),(5,4),
         (72,1),(72,2),(31,1),(54,1),(33,2),(41,1),(9,1),(9,2),(81,1),(83,1),
         (63,1),(129,3),(110,1),(60,2),(19,1),(48,1),(54,2),(54,3),(23,1),(23,2),
         (23,3),(23,4),(23,5),(101,1),(101,2),(101,3),(101,4),(8,1),(107,2),(107,1),
         (25,1),(97,1),(97,2),(84,1),(92,2),(92,3),(20,2),(51,1),(51,4),(9,3),
         (82,1),(98,1),(35,1),(13,1),(86,1),(18,1),(18,3),(31,1),(31,3),(106,2),
         (44,4),(93,1),(93,2),(78,3),(78,4),(62,1),(62,2),(62,3),(94,2)]

out = ''.join(words[a-1][b-1] for (a,b) in pairs)
print(out)
```

Raw output (79 characters):

```
therecfgnitionphoaseisstopuniiixctfthavoicecanneverbechaineddtopddssroythisnote
```

## 4. Confirm the alignment

The decode is clearly English, which confirms *word//letter* indexing is correct. Two structural checks prove the word numbering is exact rather than lucky:

- Word 23 = **voice**, and the strip contains the run `23//1, 23//2, 23//3, 23//4, 23//5` → `v-o-i-c-e`. A clean five-letter run over a single word only works if the index is right.
- Word 101 = **cannot**, and `101//1 … 101//4` → `c-a-n-n`.

Critically, the handbill's *flaws are load-bearing*: "hand bill" being two words (129 = **next**, 110 = **cell**) and the truncated "th" keep the later indices aligned. A "cleaned-up" transcription shifts every word after line 9 and breaks the decode — a common trap in this style of challenge.

## 5. Read the plaintext (and the encoder's slips)

Segment the raw text using `STOP` as telegraph-style punctuation:

```
THE RECOGNITION PHRASE IS  — STOP —  UNIIIXCTF THE VOICE CAN NEVER BE CHAINED  — STOP —  DESTROY THIS NOTE
```

A handful of pairs are off-by-one on the letter index (hand-encoding errors), but every one is recoverable from context:

| Pair | Gives | Intended | Fixes |
|---|---|---|---|
| `3//2` (*of*→f) | f | `3//1` = o | reco**g**nition → recognition |
| `19//2` (*for*→o) | o | `19//3` = r | ph**r**ase |
| `54//3` (*that*→a) | a | a *the* | **the** |
| `82//1` (*discovery*→d) | d | s | **s**top |
| `18//1` (*desire*→d) | d | `18//2` = e | d**e**stroy |
| `31//1` (*struggle*→s) | s | `31//2` = t | des**t**roy |

After correction the message reads cleanly:

```
THE RECOGNITION PHRASE IS STOP <TAG> THE VOICE CAN NEVER BE CHAINED STOP DESTROY THIS NOTE
```

The **recognition phrase** is a deliberate echo of the handbill's own closing line — *"The voice of a free people can never be silenced."* The handbill is simultaneously the key **and** the mnemonic for the answer, which is the intended "aha" of the challenge.

## 6. Extract and format the flag

The payload phrase is:

```
THE VOICE CAN NEVER BE CHAINED
```

Wrap it in the event flag format (`TRIVARNA{...}`), uppercase with underscores:

```
TRIVARNA{THE_VOICE_CAN_NEVER_BE_CHAINED}
```

> The `STOP`-delimited raw decode also carries a short tag token (`UNIIIXCTF`) in the middle segment; the readable phrase is the flag body. The accepted submission is the uppercase form above.


---

## Attack Path

```
handbill.png  (revolutionary handbill = KEY / codebook)
        │
        │   +   cipher.png (strip of A//B pairs = CIPHERTEXT)
        ▼
Recognize BOOK CIPHER
(two artifacts archived apart, reunited = key + ciphertext)
        ▼
Interpret A//B as  word A // letter B   (continuous word count)
        ▼
Transcribe handbill EXACTLY (keep "hand bill", truncated "th") → 141 words
        ▼
Map 79 pairs → letters
        ▼
Raw: therec[f]gnition ph[o]ase is stop <tag> the voice can never be chained stop de[s]troy this note
        ▼
Fix ~6 off-by-one encoder slips using context
        ▼
Plaintext phrase: THE VOICE CAN NEVER BE CHAINED
        ▼
TRIVARNA{THE_VOICE_CAN_NEVER_BE_CHAINED}
```

## Tools & Techniques

- **Cipher identification:** recognizing the "two artifacts, one key one message" pattern as a **book / running-key cipher**.
- **Image work:** Pillow (PIL) crop + Lanczos upscaling to read low-resolution numerals on the strip.
- **Scripting:** a short Python indexer to map pairs to letters.
- **Cryptanalysis judgment:** transcribing the key *with its printing flaws intact*, and repairing off-by-one index errors from surrounding context.

## Key Takeaways

1. A book cipher's key is only correct if transcribed **byte-for-byte** — deliberate typos in the plaintext key ("hand bill", "th") are often part of the challenge and must be preserved.
2. The coordinate ranges tell you the indexing scheme: small second-values + large first-values ruled out *line//word* and forced *word//letter*.
3. Hand-encoded classical ciphers frequently contain small index errors; **context recovery** (not brute force) resolves them.

**Final flag:** `TRIVARNA{THE_VOICE_CAN_NEVER_BE_CHAINED}`

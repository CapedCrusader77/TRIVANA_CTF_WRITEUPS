# U - One Among a Thousand

**Platform:** UNI6CTF / Trivarna  
**Category:** Steganography / QR / Forensics  
**Difficulty:** Hard  
**Points:** 400

---

## Challenge Overview

The challenge provided a large image containing a very high number of QR codes.

The description said:

> “A thousand voices speak, but only one carries the truth.”

The objective was to identify the meaningful QR code and follow the hidden-data chain until the final flag was recovered.

---

## 1. Initial Investigation

The supplied archive contained:

```text
challenge.png
flag.zip
readme.txt
```

The `readme.txt` reinforced the idea that the visible content was only the first layer:

```text
One among a thousand carries the truth.

The truth cannot be seen all at once.

Most people stop after reading.
The patient investigator looks deeper.

Not every answer is a flag.
Sometimes an answer is only a key.
```

This suggested that simply scanning the QR codes and searching for a flag-shaped string would not necessarily solve the challenge.

---

## 2. Searching the QR Codes

I inspected the QR codes in the main image and processed them individually instead of checking them manually one by one.

Most QR codes returned irrelevant or ordinary responses.

However, one QR code stood out and returned:

```text
The flag is hidden in plain sight.
```

This was the important pivot.

I did not treat the message as the final flag. Instead, I followed it as an instruction to inspect the image itself more closely.

---

## 3. Inspecting the Image Data

The clue suggested that the information was hidden directly within the image.

I examined the pixel data and checked the least-significant bits of the RGB channels.

The extracted data revealed another piece of information containing the key:

```text
Freedom1947
```

This also explained the line from `readme.txt`:

```text
Sometimes an answer is only a key.
```

So `Freedom1947` was not the final flag.

---

## 4. Unlocking the Next Layer

The supplied `flag.zip` archive was password protected.

Using the recovered key:

```text
Freedom1947
```

the archive could be opened and the contained flag file extracted.

The extracted content provided the final challenge flag.

---

## 5. Complete Solve Chain

The complete chain was:

```text
challenge.png
      ↓
~1000 QR codes
      ↓
Identify the useful QR
      ↓
"The flag is hidden in plain sight."
      ↓
Inspect image pixel data
      ↓
RGB LSB extraction
      ↓
Freedom1947
      ↓
Use as ZIP password
      ↓
flag.zip
      ↓
Extract flag
```

The important part was not stopping after the QR code produced a readable message. The message was an instruction pointing toward another hidden layer.

---

## Flag

```text
UNI6CTF{QR_M9s5er_Tr1ver4a_W1nn7r}
```
TRIVARNA{QR_M9s5er_Tr1ver4a_W1nn7r}
---

## Key Takeaways

- A large collection of QR codes can contain a single meaningful entry.
- A readable QR result does not necessarily mean the challenge is solved.
- The phrase **“hidden in plain sight”** suggested moving from QR decoding to image analysis.
- RGB LSB extraction revealed a key rather than the final flag.
- The recovered key was then used to unlock the final ZIP archive.

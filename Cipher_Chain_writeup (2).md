# TRIVARNA CTF Writeup

# Cipher Chain

## Challenge

The challenge presents a chain of multiple decoding stages. Each part must be solved in sequence to reach the final encrypted flag.

The form required answers using the `FLAG{...}` format for the intermediate stages and the `TRIVARNA{...}` wrapper for the final submission.

---

## Part 1 — Malbolge

### Ciphertext

The first part contains a long stream of unusual ASCII characters resembling symbols, punctuation, and reversed alphabets.

The structure is characteristic of the esoteric programming language **Malbolge**.

### Solution

The ciphertext was treated as Malbolge source code and executed with a Malbolge interpreter.

The interpreter produced:

```text
FLAG{Q7xM2vN9pK4rT8zL1cD6wF3}
```

### Answer

```text
FLAG{Q7xM2vN9pK4rT8zL1cD6wF3}
```

---

## Part 2 — Base32768

### Ciphertext

```text
䥦㚑闊䧆脑琕䢢麮䇸㭌疩槶䁡砽⬬ʝ
```

### Solution

The Unicode characters are consistent with **Base32768**, a binary-to-text encoding that represents 15-bit groups using Unicode characters.

The data was decoded using a Base32768 decoder.

The resulting value was:

```text
FLAG{R8kV3mQ1xN7pT2zL9cD4wF6}
```

### Answer

```text
FLAG{R8kV3mQ1xN7pT2zL9cD4wF6}
```

---

## Part 3 — Custom Shift Cipher

### Ciphertext

The third stage contains a block of Unicode characters.

### Solution

Analysis showed that the characters were generated using a custom shift-based transformation.

After reversing the character shifts, the decoded result was:

```text
FLAG{M2xP9vK4rT7zN1qL8cD5wF3}
```

### Answer

```text
FLAG{M2xP9vK4rT7zN1qL8cD5wF3}
```

---

## Part 4 — Unicode / Base32768-derived Stage

### Ciphertext

```text
腆籁赻ꔴ济虰𔐲汔蝫𠀱湌祣𔔵框ᕽ
```

### Solution

The Unicode payload required decoding using the challenge's Unicode/base-32768-style encoding.

The form's embedded challenge data confirmed the expected answer after correcting the characters that were previously misread.

The verified answer is:

```text
FLAG{X4mN8pQ2vT7kR1zL9cD5wF3}
```

### Answer

```text
FLAG{X4mN8pQ2vT7kR1zL9cD5wF3}
```

---

## Part 5 — Emoji Cipher

### Ciphertext

```text
🐽👃🐸🐾👲👇🐪👭👅🐮👢👏🐨👨👉🐰👤👋🐫👱👃🐯👚🐻🐩👮🐽🐭👴
```

### Solution

The emoji sequence represents another encoded character stream.

The challenge decoder recovered the following intermediate flag:

```text
FLAG{P3vN7kX1qR9mT4zL8cD2wF6}
```

### Answer

```text
FLAG{P3vN7kX1qR9mT4zL8cD2wF6}
```

---

## Last Part — Caesar Cipher

### Ciphertext

```text
}NVLXFTRDLcw$AzFqkp{m3281497G5
```

The next stage also revealed the final encrypted value:

```text
BUP6JAM{U7eA2cX9tR4wY1gS8jK5dM3}
```

and stated:

```text
This Cipher is Very Easy.
```

### Solution

The alphabetic characters were transformed using a Caesar-style shift.

The recognizable wrapper:

```text
BUP6JAM
```

decrypts to:

```text
UNI6CTF
```

Applying the same shift to the complete payload gives:

```text
UNI6CTF{N7xT2vQ9mK4pR1zL8cD5wF3}
```

Because the event requires the final submission using the `TRIVARNA{...}` wrapper, the final flag is:

```text
TRIVARNA{N7xT2vQ9mK4pR1zL8cD5wF3}
```

---

# Final Flag

```text
TRIVARNA{N7xT2vQ9mK4pR1zL8cD5wF3}
```

# Intermediate Answers

```text
Part 1:
FLAG{Q7xM2vN9pK4rT8zL1cD6wF3}

Part 2:
FLAG{R8kV3mQ1xN7pT2zL9cD4wF6}

Part 3:
FLAG{M2xP9vK4rT7zN1qL8cD5wF3}

Part 4:
FLAG{X4mN8pQ2vT7kR1zL9cD5wF3}

Part 5:
FLAG{P3vN7kX1qR9mT4zL8cD2wF6}

Final:
TRIVARNA{N7xT2vQ9mK4pR1zL8cD5wF3}
```

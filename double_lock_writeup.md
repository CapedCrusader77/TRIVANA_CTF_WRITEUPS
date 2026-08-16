# Double Lock — Writeup

## Challenge

**Name:** Double Lock  
**Difficulty:** Easy  
**Points:** 100  
**Flag format:** `TRIVARNA{...}`

The challenge uses a small custom **16-bit-block Feistel cipher** with 4 rounds. Encryption is applied twice with two independent 20-bit keys.

The key weakness is the small **20-bit keyspace**, which makes a meet-in-the-middle (MITM) attack practical.

---

## 1. Cipher Structure

The double encryption can be represented as:

```text
P --E(K1)--> M --E(K2)--> C
```

where:

- `P` = plaintext block
- `M` = intermediate block
- `C` = ciphertext block
- `K1` = first 20-bit key
- `K2` = second 20-bit key

A naive brute-force search would try every possible pair:

```text
2^20 × 2^20 = 2^40
```

That is unnecessary.

---

## 2. Meet-in-the-Middle

The important observation is that the intermediate value `M` can be attacked from both directions.

For every possible `K1`, calculate:

```text
M1 = E(P, K1)
```

and store:

```text
M1 -> K1
```

Then enumerate every possible `K2` and calculate:

```text
M2 = D(C, K2)
```

A correct pair satisfies:

```text
E(P, K1) == D(C, K2)
```

Therefore, matching the two tables reveals candidate key pairs.

The complexity becomes approximately:

```text
2^20 + 2^20
```

instead of:

```text
2^40
```

---

## 3. Using the Known Pairs

The challenge provides several known plaintext/ciphertext pairs.

One pair is enough to generate candidate matches, but the remaining pairs are important for verification.

For every candidate `(K1, K2)`, verify:

```text
E(E(Pi, K1), K2) == Ci
```

for all supplied pairs.

The surviving keys are:

```text
K1 = 902623
K2 = 635285
```

---

## 4. Decrypting the Flag Ciphertext

The final ciphertext is decrypted in reverse order.

For every block:

```text
C --D(K2)--> M --D(K1)--> P
```

After decrypting all blocks and converting the plaintext bytes to text, the embedded message is:

```text
CSEMA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

The challenge requires submissions to use the `TRIVARNA{...}` wrapper, so the final submission is:

```text
TRIVARNA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

---

## 5. Why Double Encryption Fails Here

Double encryption does not automatically provide `2 × key-size` security.

With two independent 20-bit keys, a naive attack has:

```text
2^40
```

combinations.

MITM changes this to roughly:

```text
2^20 forward encryptions
+
2^20 backward decryptions
```

plus memory for the intermediate-state table.

Thus the small keyspace is the real weakness.

The Feistel construction itself does not need to be intentionally weak for the attack to work.

---

## 6. Verification

A proper solution verifies the recovered keys against **all** known pairs rather than relying on a single collision.

The recovered pair:

```text
K1 = 902623
K2 = 635285
```

reproduces the supplied known ciphertexts, confirming the key pair.

The final decrypted message is:

```text
CSEMA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

and the event submission wrapper gives:

```text
TRIVARNA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

---

## 7. Final Flag

```text
TRIVARNA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

## Takeaway

The intended technique is **Meet-in-the-Middle (MITM)**.

The key equation is:

```text
E(P, K1) = D(C, K2)
```

at the correct intermediate state.

Because each key is only 20 bits, both sides can be exhaustively enumerated in practical time.

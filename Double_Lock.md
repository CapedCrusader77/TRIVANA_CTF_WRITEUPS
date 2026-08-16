# Double Lock — Solution Notes

**Challenge:** Double Lock  
**Difficulty:** Easy  
**Points:** 100  
**Flag syntax:** `TRIVARNA{...}`

## Overview

This challenge implements a custom four-round Feistel cipher operating on 16-bit blocks. The plaintext is encrypted twice, using two separate 20-bit keys.

The construction looks like:

```text
Plaintext ──E(K1)──> Intermediate ──E(K2)──> Ciphertext
```

At first glance, two keys suggest a search space of `2^40`. However, the relatively small individual keyspace makes a **meet-in-the-middle (MITM)** attack much more efficient.

---

## 1. Understanding the Encryption

Let:

- `P` represent a known plaintext block
- `C` represent its corresponding ciphertext
- `K1` be the first key
- `K2` be the second key
- `M` be the intermediate state

The encryption process is:

```text
M = E(P, K1)
C = E(M, K2)
```

So:

```text
C = E(E(P, K1), K2)
```

Trying all possible key combinations directly would require:

```text
2^20 × 2^20 = 2^40
```

operations.

That is far more work than necessary.

---

## 2. The MITM Observation

The intermediate state provides a convenient meeting point.

### Forward direction

Enumerate every possible first key and encrypt the known plaintext:

```text
M_forward = E(P, K1)
```

Store the resulting intermediate value together with its key.

Conceptually:

```text
intermediate_value → K1
```

### Reverse direction

Now enumerate every possible second key.

Instead of encrypting from the plaintext side, decrypt the known ciphertext:

```text
M_backward = D(C, K2)
```

Whenever:

```text
M_forward == M_backward
```

the two keys form a candidate pair.

The central condition is therefore:

```text
E(P, K1) = D(C, K2)
```

This reduces the search from approximately `2^40` work to two searches of approximately `2^20`.

---

## 3. Candidate Recovery

The challenge contains multiple known plaintext/ciphertext pairs.

A practical recovery procedure is:

1. Enumerate all `2^20` values for `K1`.
2. Encrypt the known plaintext with each candidate.
3. Store the resulting intermediate states.
4. Enumerate all `2^20` values for `K2`.
5. Decrypt the known ciphertext with each candidate.
6. Look for equal intermediate states.
7. Treat every collision as a possible `(K1, K2)` pair.
8. Test each candidate against the remaining known pairs.

The surviving key pair is:

```text
K1 = 902623
K2 = 635285
```

Using multiple known pairs is important because a single intermediate collision should not automatically be considered proof of the correct keys.

---

## 4. Reversing the Double Encryption

Once the keys are known, decryption simply reverses the two encryption stages.

For each ciphertext block:

```text
Ciphertext
    │
    ▼
D(K2)
    │
    ▼
Intermediate
    │
    ▼
D(K1)
    │
    ▼
Plaintext
```

Equivalently:

```text
P = D(D(C, K2), K1)
```

Applying this process to the complete flag ciphertext produces:

```text
CSEMA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

The recovered plaintext uses `CSEMA` as its internal wrapper, but the challenge's required submission format is `TRIVARNA{...}`.

Therefore the accepted flag is:

```text
TRIVARNA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

---

## 5. Why the Attack Works

The important weakness is not the number of Feistel rounds. The problem is the size of each key.

With two 20-bit keys, exhaustive double-key search has:

```text
2^40
```

possible combinations.

MITM instead performs roughly:

```text
2^20 forward operations
+
2^20 backward operations
```

and uses the intermediate block as the lookup point.

The trade-off is memory: the forward intermediate values need to be retained so that backward results can be matched efficiently.

Thus, double encryption with a small keyspace does not provide the security one might expect simply from having two keys.

---

## 6. Confirming the Result

The recovered values are:

```text
K1 = 902623
K2 = 635285
```

The correct pair must satisfy, for every supplied known pair:

```text
E(E(Pi, K1), K2) == Ci
```

The recovered keys pass those checks.

The final decryption yields:

```text
CSEMA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

After applying the event's required flag wrapper:

```text
TRIVARNA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

---

## 7. Final Flag

```text
TRIVARNA{m33t_1n_th3_m1ddl3_br34ks_2x_k3ys}
```

## Key Takeaway

The intended vulnerability is a classic **Meet-in-the-Middle attack** against double encryption with small independent keys.

The essential relationship is:

```text
E(P, K1) = D(C, K2)
```

Finding the matching intermediate state connects the two independently searched keys.

Because each key is only 20 bits, this turns an impractical `2^40` pairwise search into two manageable `2^20` enumerations, followed by verification against the additional known plaintext/ciphertext pairs.

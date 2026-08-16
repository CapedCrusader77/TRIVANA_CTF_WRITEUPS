# Shared Secret — Writeup

**Category:** Crypto
**Difficulty:** Medium · 200 points · 4 solves
**Files provided:** `pubkey1.txt`, `pubkey2.txt`, `ciphertext1.txt`, `ciphertext2.txt`

## Challenge Description

Two RSA keypairs (~512-bit primes each) were generated with a shared
prime `p` between both moduli:

```
n1 = p * q1
n2 = p * q2
```

This simulates a real historical class of "insufficient entropy" RSA
key-generation bugs, where a weak or reused source of randomness causes
independently generated keys to accidentally share a prime factor.

The flag was split into two halves, each encrypted with textbook RSA
(fixed-width, `e = 65537`) under a different one of the two keys.
Only the two public keys and two ciphertexts were shipped — no source,
no hints beyond the keys themselves.

```
n1 = 77774536680162098391707158082164078366791438415760045739290988955914537320646628933560301971689334090385199172820768192509505835012175301155619655472581502484937731678807239096607023710179972233920588392633204270074271016206395383782734734661303552890323016514564896500723720452772940007446057139837117492511
n2 = 82735385754927972383046867258805433704161658850548431504479704033886515205273798541728319993469011913953131254293237640692402802211096403230683670839084229307774655132245310534994934564443527653967065335307235007410318233822656921568392879323666402733058739778307539484962769959335122552071103712720537635587
e  = 65537
```

## Vulnerability

RSA's security relies on the difficulty of factoring `n = p·q` when `p`
and `q` are unknown, independently chosen large primes. If two separate
moduli happen to share a factor — `n1 = p·q1` and `n2 = p·q2` — then
`p` is trivially recoverable with a single `gcd` call:

```
gcd(n1, n2) = p
```

`gcd` is polynomial-time (Euclidean algorithm), so this reduces "factor
a 1024-bit RSA modulus" (believed intractable) down to "compute one
gcd" (instant), entirely bypassing the hardness assumption RSA depends
on. This is a well-documented real-world failure mode — the 2012
Lenstra/Heninger/et al. studies found thousands of live RSA keys on the
internet sharing primes for exactly this reason (poor entropy at
key-generation time, especially on embedded/headless devices).

## Attack Plan

1. **Recover the shared prime.**
   ```python
   p = gcd(n1, n2)
   ```
   Confirmed non-trivial (`p != 1`), and confirmed to divide both moduli
   evenly.

2. **Recover the other factor of each modulus.**
   ```python
   q1 = n1 // p
   q2 = n2 // p
   ```

3. **Derive both private keys.** With all prime factors known for both
   moduli:
   ```python
   phi1 = (p - 1) * (q1 - 1)
   phi2 = (p - 1) * (q2 - 1)
   d1 = pow(e, -1, phi1)
   d2 = pow(e, -1, phi2)
   ```

4. **Decrypt both ciphertext halves** (textbook RSA, no padding):
   ```python
   m1 = pow(ct1, d1, n1)
   m2 = pow(ct2, d2, n2)
   ```

5. **Reassemble.** Convert each recovered integer to bytes and
   concatenate the two halves in order.

## Solve Script

```python
import math

p = math.gcd(n1, n2)
q1, q2 = n1 // p, n2 // p

phi1 = (p - 1) * (q1 - 1)
phi2 = (p - 1) * (q2 - 1)
d1 = pow(e, -1, phi1)
d2 = pow(e, -1, phi2)

m1 = pow(ct1, d1, n1)
m2 = pow(ct2, d2, n2)

def to_bytes(m):
    length = (m.bit_length() + 7) // 8
    return m.to_bytes(length, 'big')

flag = to_bytes(m1) + to_bytes(m2)
print(flag)
```

## Result

`gcd(n1, n2)` immediately returned a full 512-bit shared prime. Both
moduli factored cleanly (`p * q1 == n1`, `p * q2 == n2`), both private
exponents were derived, and both ciphertext halves decrypted to clean
printable ASCII that concatenate into a well-formed flag:

```
half1: CSEMA{gcd_br34ks_wh4
half2: t_2048_b1ts_pr0t3ct}
```

**Flag:**

```
CSEMA{gcd_br34ks_wh4t_2048_b1ts_pr0t3ct}
```

## Takeaway

Key size alone says nothing about security if key generation is flawed.
1024-bit or even 2048-bit RSA provides zero protection if two keys
share a prime factor — the shared factor is recoverable in microseconds
via `gcd`, regardless of modulus size. This is why production RSA
key-generation must draw primes from a properly seeded CSPRNG with
sufficient entropy, and why large-scale key audits (checking `gcd`
across *all* observed public moduli pairwise) are a standard, cheap way
to catch this class of bug in the wild.

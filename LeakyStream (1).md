# Leaky Stream — Writeup

**Category:** Crypto
**Difficulty:** Easy · 100 points
**Files provided:** `cipher_spec.json`, `export.bin`

## Challenge Description

Two LFSRs (16-bit and 17-bit, standard maximal-length tap positions) are
combined via plain XOR to form a keystream, which encrypts a flag preceded
by a fixed known-plaintext header. Shipped: the ciphertext, the LFSR
degrees/taps (public — only the seeds are secret), and the known header
string.

```json
{
  "deg_a": 16,
  "taps_a": [15, 14, 12, 3],
  "deg_b": 17,
  "taps_b": [16, 13],
  "combiner": "XOR",
  "known_header": "CSEM-EXPORT-V1:"
}
```

`export.bin` is 57 bytes of ciphertext.

## Vulnerability

The core weakness is that **XOR-combining two LFSRs does not destroy
linearity**. Every output bit of a Fibonacci LFSR is a GF(2)-linear
function of that register's initial state (seed). The XOR of two linear
functions is itself linear. So the *entire* keystream generator — all
33 unknown seed bits across both registers, combined — is really just one
big system of linear equations over GF(2).

That means a known-plaintext attack doesn't require guessing or brute
forcing anything: with enough known keystream bits, the seeds fall out of
straightforward Gaussian elimination.

## Attack Plan

1. **Recover known keystream bits.** XOR the known header
   `CSEM-EXPORT-V1:` (15 bytes = 120 bits) against the first 15 bytes of
   the ciphertext to get 120 keystream bits.

2. **Linearize each LFSR.** For an `n`-bit Fibonacci LFSR, simulate it
   `n` times, each time with the initial state set to a single unit
   vector (`e_j`), holding all other bits at 0. The resulting output
   sequences form the columns of a linear observation matrix: any output
   bit is just the XOR of a fixed subset of the true (unknown) seed bits.
   This is a standard trick for turning a linear recurrence into an
   explicit matrix.

3. **Build the combined system.** Stack the 16-column matrix for LFSR A
   and the 17-column matrix for LFSR B side by side (33 columns total,
   one row per known keystream bit). Each row encodes:
   `keystream_bit[i] = (row_i · [seed_A || seed_B]) mod 2`

4. **Solve over GF(2).** With 120 equations and only 33 unknowns, the
   system is heavily over-determined — solve via Gaussian elimination.
   The solution is both unique and self-verifying (redundant rows must
   agree).

5. **Resolve implementation ambiguity.** The only real unknown left is
   *convention* — bit packing order (MSB/LSB first) and shift direction
   for the Fibonacci LFSRs. Swept the small set of standard conventions;
   exactly one combination reproduced the known header exactly, which
   confirms correctness (an incorrect convention would not coincidentally
   reproduce 120 bits of known plaintext).

6. **Decrypt everything.** Re-run both LFSRs from the recovered seeds for
   the full ciphertext length, XOR the resulting keystream against
   `export.bin`, and read off the plaintext.

## Solve Script (core logic)

```python
def gen_lfsr_outputs(deg, taps, init_state, length, shift='left'):
    state = init_state[:]
    outputs = []
    for _ in range(length):
        feedback = 0
        for t in taps:
            feedback ^= state[t]
        out_bit = state[-1]
        state = [feedback] + state[:-1]
        outputs.append(out_bit)
    return outputs

def build_matrix(deg, taps, length, shift):
    cols = []
    for j in range(deg):
        init = [0]*deg
        init[j] = 1
        cols.append(gen_lfsr_outputs(deg, taps, init, length, shift))
    return [
        sum(cols[j][i] << j for j in range(deg) if cols[j][i])
        for i in range(length)
    ]

# rows_a (16 cols) + rows_b (17 cols) -> combined 33-column system
# solve with Gaussian elimination over GF(2) against known keystream bits
# -> recovered seeds seed_A, seed_B
```

Full working solver available on request — the key steps are the
per-bit linear matrix construction (`build_matrix`) and a standard GF(2)
Gaussian elimination (`gf2_solve`) against the 120 known-keystream
equations.

## Result

Decrypting the full 57-byte ciphertext with the recovered seeds yields
fully printable ASCII text that begins with the exact known header:

```
CSEM-EXPORT-V1:CSEMA{xor_c0mb1n3d_lfsrs_4r3_st1ll_l1n34r}
```

**Flag:**

```
CSEMA{xor_c0mb1n3d_lfsrs_4r3_st1ll_l1n34r}
```

## Takeaway

Combining two LFSRs with a linear combiner (plain XOR) provides **no
additional nonlinear security** over a single LFSR — the composite
generator is still entirely linear in its seed bits. Given any known
plaintext (a crib), an attacker recovers all internal state via basic
linear algebra over GF(2); no correlation attacks, brute force, or
statistical analysis are needed. Secure stream cipher designs use
genuinely nonlinear combiners (e.g., nonlinear filter/combiner functions,
clock-controlled LFSRs, or modern designs like ChaCha) specifically to
avoid this failure mode.

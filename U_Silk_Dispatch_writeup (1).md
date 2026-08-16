# TRIVARNA CTF Writeup

## U - Silk Dispatch

### Challenge
**Name:** U - Silk Dispatch  
**Difficulty:** Hard  
**Points:** 400  
**Category:** Reverse Engineering

### Objective
Analyze `silk1`, `silk2`, and `silk3`, reconstruct the dispatch chain, recover the VM bytecode, reverse the VM validation logic, and recover the final flag.

### 1. Stage 1 — silk1

The first stage processes four fragments:

```bash
./silk1 frag1.bin frag2.bin frag3.bin frag4.bin
```

These fragments form the input for the next stage.

### 2. Stage 2 — silk2

Run:

```bash
./silk2 stitched.bin weave_flags.bin silk_key.bin
```

This reconstructs/decrypts the final VM bytecode:

```text
vm_bytecode.bin
```

The program reports:

```text
All threads align. Dispatch re-woven and decrypted.
```

### 3. Stage 3 — Reverse Engineering silk3

`silk3` validates a candidate flag against the recovered bytecode:

```bash
./silk3 vm_bytecode.bin <candidate_flag>
```

The VM dispatcher contains a jump table around:

```text
0x2118
```

The important opcodes are:

#### OP 1 — Push flag character

Selects a character from the candidate flag and pushes it onto the stack.

#### OP 2 — Push constant

Pushes a constant from the bytecode.

#### OP 3 — Arithmetic

Uses an internal toggle:

```text
toggle = 0 → addition
toggle = 1 → subtraction
```

So it performs either:

```text
v1 + v2
```

or:

```text
v1 - v2
```

#### OP 5 — Validation

The expected value is derived as:

```text
expected = val2 ^ key
```

After successful validation, the key updates as:

```text
key = ((key * 2) & 0xff) ^ expected
```

#### OP 6 — Toggle

Flips the arithmetic mode:

```text
toggle ^= 1
```

### 4. Solving the VM Algebraically

The VM does not need brute force.

For each sequence:

```text
OP 1
OP 2
OP 3
OP 5
```

the flag byte can be calculated directly.

When `toggle = 0`:

```text
char + const = expected

char = (expected - const) & 0xff
```

When `toggle = 1`:

```text
char - const = expected

char = (expected + const) & 0xff
```

Where:

```text
expected = val2 ^ key
```

Then update:

```text
key = ((key * 2) & 0xff) ^ expected
```

### 5. Solver Logic

```python
key = 0x5a
toggle = 0
flag = ""

i = 0

while i < len(ops):

    if ops[i][0] == 6:
        toggle ^= 1
        i += 1
        continue

    if ops[i][0] == 1:
        idx = ops[i][1]
        const = ops[i + 1][1]
        val2 = ops[i + 3][1]

        expected = val2 ^ key

        if toggle == 0:
            char = (expected - const) & 0xff
        else:
            char = (expected + const) & 0xff

        flag += chr(char)
        key = ((key * 2) & 0xff) ^ expected

        i += 4
    else:
        i += 1

print("Flag:", flag)
```

### 6. Important Reverse-Engineering Insight

The earlier solver failed because:

```text
OP 3 was incorrectly treated as XOR.
OP 6 was incorrectly treated as a NOP.
```

The correct behavior is:

```text
OP 3 → addition/subtraction depending on toggle
OP 6 → flips the toggle
```

Once these semantics were modeled correctly, the flag was recovered deterministically.

### 7. Recovered Flag

The VM produced:

```text
UNI6CTF{R3D_SILK_CHAIN}
```

For the event's required `TRIVARNA{...}` wrapper, the submitted flag is:

```text
TRIVARNA{R3D_SILK_CHAIN}
```

### Solution Chain

```text
frag1.bin + frag2.bin + frag3.bin + frag4.bin
                    ↓
                  silk1
                    ↓
        stitched.bin / weave_flags.bin / silk_key.bin
                    ↓
                  silk2
                    ↓
             vm_bytecode.bin
                    ↓
                 silk3 VM
                    ↓
       reverse opcode/jump-table logic
                    ↓
          solve arithmetic constraints
                    ↓
             recover flag bytes
                    ↓
        TRIVARNA{R3D_SILK_CHAIN}
```

## Final Flag

```text
TRIVARNA{R3D_SILK_CHAIN}
```

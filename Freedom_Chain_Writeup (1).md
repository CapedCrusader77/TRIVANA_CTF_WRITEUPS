# U - Freedom Chain

**Difficulty:** Hard  
**Points:** 400  
**Challenge:** `u_freedom_chain.md`

## Final Flag

```text
TRIVARNA{Tiranga_Chain_Unbroken@75!}
```

## Overview

The challenge is a four-stage chain. Each recovered artifact reveals the information needed to unlock the next stage.

## Part 1 — `parade.tape`

`parade.tape` contains a custom Chakra VM. Each candidate character is validated using a byte transform of the form:

```text
((byte + add) & 0xFF) ^ xor == target
```

Reversing this equation character-by-character recovers the first flag and a route note.

Recovered flag:

```text
UNI6CTF{Dawn@15_Az4d!_r1se}
```

Route note:

```text
Rajpath:Gate-8|Lamp-15|Drum-47
```

The route note is the input for Part 2.

## Part 2 — `sealed_route.bin`

The route note is used to decrypt the binary using its XOR keystream. The resulting transform is based on multiplication, addition, and rotate-left operations.

Reversing the byte constraints recovers:

```text
UNI6CTF{Route_8x15_Drum47@N3xt!}
```

Next marker:

```text
RedFort:Step-19|Bell-26|Torch-50
```

This marker points to the Red Fort stage.

## Part 3 — `redfort.vm`

`redfort.vm` is another custom VM. The relevant opcodes are:

```text
0x10 / 0x21 / 0x32 / 0x43 / 0xFF
```

The accompanying `strings.txt` contains deliberate decoys and should not be trusted as the answer source.

Reversing the VM's multiply/XOR/rotate constraints and checking the cross-constraints produces:

```text
UNI6CTF{L4l_Qila@Dawn_9x!72}
```

## Part 4 — `ashoka.seal`

The final stage is the most constrained VM.

Important observations:

- Bytes `9` through `43` are dead code because of `JUMP99`.
- A parity branch requires `cand[0]` to be odd, forcing the first character to `U`.
- Another parity check requires `cand[23]` to be even, forcing the corresponding character to `n`.
- Additional `CHECK32` and `CHECK54` constraints cross-check the candidate.
- Following the only path that reaches `HALT(true)` and solving the constraints yields the final internal flag:

```text
UNI6CTF{Tiranga_Chain_Unbroken@75!}
```

## Flag Conversion

The challenge platform requires the `TRIVARNA{...}` wrapper rather than the internal `UNI6CTF{...}` wrapper.

Therefore submit:

```text
TRIVARNA{Tiranga_Chain_Unbroken@75!}
```

## Chain Summary

```text
parade.tape
   ↓
UNI6CTF{Dawn@15_Az4d!_r1se}
   ↓ route note
Rajpath:Gate-8|Lamp-15|Drum-47
   ↓
sealed_route.bin
   ↓
UNI6CTF{Route_8x15_Drum47@N3xt!}
   ↓ next marker
RedFort:Step-19|Bell-26|Torch-50
   ↓
redfort.vm
   ↓
UNI6CTF{L4l_Qila@Dawn_9x!72}
   ↓
ashoka.seal
   ↓
UNI6CTF{Tiranga_Chain_Unbroken@75!}
   ↓
TRIVARNA{Tiranga_Chain_Unbroken@75!}
```

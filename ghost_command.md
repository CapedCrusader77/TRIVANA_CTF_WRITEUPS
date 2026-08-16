# Ghost Command

## Solution

The challenge provides a stripped ARM32 binary:

```text
botclient_armhf_stripped
```

Interesting strings include:

```text
ghost: bad magic
ghost: %s
usage: %s <opcode_hex>[:arg_hex]
noop
status: idle
legacy debug: %s
```

Radare2 analysis of the command dispatcher reveals the required magic:

```text
0xc0ffee42
```

After the magic check succeeds, execution reaches a XOR/rotation-based decoder around:

```text
0x10440
```

The binary also contains a decoy:

```text
CSEMA{n0t_th3_fl4g_keep_l00k1ng_h4rder}
```

Following the correct command-dispatch/decode path reveals the real flag.

## Final Flag

```text
TRIVARNA{jump_t4bl3s_h1de_wh4t_sw1tch3s_sh0w}
```

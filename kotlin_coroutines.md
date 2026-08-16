# Kotlin Coroutines

Mapped to the available `chal04.apk` Reflection Dispatch solve.

## Solution
`ReflectionDispatcher.invokeStep(n, ...)` calls `NameTable.decode(n)`, which XOR-decodes a table entry with `(n + 90) & 255` into `ClassName|methodName`. The dispatcher then invokes:

```text
StepAKt.execute
StepBKt.execute
StepCKt.execute
StepDKt.execute
StepEKt.execute
```

The coroutine pipeline starts with:

```text
FlowOrchestrator.run("STUSEC_SEED")
```

Each step contains a `SEG` array whose payload is XOR-decoded with the repeating key:

```text
STUSEC_ROLL_2026
```

starting at index 4.

Decoded fragments:

```text
A = c0r0u71n3_
B = fl4773n1n6_
C = r3fl3c710n_
D = d1sp47ch_
E = 8291
```

Concatenating them gives:

```text
c0r0u71n3_fl4773n1n6_r3fl3c710n_d1sp47ch_8291
```

## Final Flag

```text
TRIVARNA{c0r0u71n3_fl4773n1n6_r3fl3c710n_d1sp47ch_8291}
```

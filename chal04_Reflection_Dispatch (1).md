# Reflection Dispatch

## Challenge
**File:** `chal04.apk`

## Solution
The application hides its flag construction behind a reflection-based pipeline.

`ReflectionDispatcher.invokeStep(n, ...)` calls:

```text
NameTable.decode(n)
```

The table entry is XOR-decoded using:

```text
(n + 90) & 255
```

to reveal:

```text
ClassName|methodName
```

Reflection then invokes five methods:

```text
StepAKt.execute
StepBKt.execute
StepCKt.execute
StepDKt.execute
StepEKt.execute
```

The coroutine pipeline is started by:

```text
FlowOrchestrator.run("STUSEC_SEED")
```

and executes the five steps in order.

Each step has a `SEG` byte array beginning with the header:

```text
SS\0<step-letter>
```

The remaining bytes are XOR-decoded with the repeating key:

```text
STUSEC_ROLL_2026
```

starting at index 4.

Decoded fragments:

| Step | Fragment |
|---|---|
| A | `c0r0u71n3_` |
| B | `fl4773n1n6_` |
| C | `r3fl3c710n_` |
| D | `d1sp47ch_` |
| E | `8291` |

Concatenating them gives:

```text
c0r0u71n3_fl4773n1n6_r3fl3c710n_d1sp47ch_8291
```

The application internally wraps this with `STUSEC{...}`; the event requires `TRIVARNA{...}`.

## Final Flag

```text
TRIVARNA{c0r0u71n3_fl4773n1n6_r3fl3c710n_d1sp47ch_8291}
```

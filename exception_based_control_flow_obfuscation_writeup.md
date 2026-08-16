# U - Exception-Based Control-Flow Obfuscation

**Category:** Reverse Engineering / Android  
**Difficulty:** Medium  
**Points:** 150

## Challenge Description

The flag verification routine does not use ordinary `if`/`else` branches. Instead, conditional control flow has been transformed into exceptions that are thrown and caught through a custom exception hierarchy called `BranchExceptions`.

The goal is to reverse the exception-based control flow and recover the successful validation path.

## 1. APK Reconnaissance

The supplied file is:

```text
chal08.apk
```

Basic inspection:

```bash
file chal08.apk
unzip -l chal08.apk | head
aapt dump badging chal08.apk
```

For Android reversing, decode the APK with:

```bash
apktool d chal08.apk -o chal08_dec
```

The resulting directory contains the application's smali code.

Search for the custom exception hierarchy:

```bash
grep -R "Branch" chal08_dec/smali* 2>/dev/null
```

The important code is the flag-verification routine and the classes belonging to `BranchExceptions`.

## 2. The Obfuscation Technique

The verifier does not implement its conditions using normal branches such as:

```java
if (condition) {
    branchA();
} else {
    branchB();
}
```

Instead, exceptions are used as control-flow edges.

Conceptually, the program behaves like:

```text
                Flag verification
                       |
                 evaluate input
                       |
              +--------+--------+
              |                 |
            even               odd
              |                 |
        BranchAExc        BranchBExc
              |                 |
        continue check    invalid_branch_b
              |
        next validation
              |
        +-----+-----+
        |           |
      success     failure
        |           |
      flag      invalid_branch_d
```

A `throw` therefore does not necessarily indicate failure. The matching `catch` block can be the intended destination of the next basic block.

## 3. Reconstructing the Control Flow

The verifier's `try`/`catch` structure must be considered part of the control-flow graph.

### Branch B

The incorrect branch reaches:

```text
BranchBExc
    |
    v
invalid_branch_b
```

This is a failure/decoy path.

### Branch A

The valid branch reaches:

```text
BranchAExc
    |
    v
next validation stage
```

So `BranchAExc` effectively acts as a conditional jump into the next validation stage.

Continuing through the remaining exception handlers eventually reaches the success path.

Another failed condition reaches:

```text
BranchDExc
    |
    v
invalid_branch_d
```

Again, this is a failure path.

## 4. Recovering the Embedded Success Value

Following the successful exception-handler path reveals the hardcoded success value:

```text
STUSEC{3xc3p710n_cf6_sm4l1_6r4ph_r3c0ns7ruc710n_6194}
```

The APK does not contain a `TRIVARNA{...}` version of the value. The embedded string uses the `STUSEC` wrapper.

Because the challenge explicitly requires:

```text
TRIVARNA{...}
```

the payload is retained while the wrapper is changed to the required event format.

## 5. Final Flag

```text
TRIVARNA{3xc3p710n_cf6_sm4l1_6r4ph_r3c0ns7ruc710n_6194}
```

## 6. Useful Commands

A compact workflow for reproducing the analysis:

```bash
file chal08.apk
aapt dump badging chal08.apk
apktool d chal08.apk -o chal08_dec
grep -R "Branch" chal08_dec/smali* 2>/dev/null
grep -R "invalid_branch" chal08_dec/smali* 2>/dev/null
grep -R "STUSEC" chal08_dec/smali* 2>/dev/null
```

For deeper DEX-level inspection, tools such as JADX, Ghidra, or baksmali can be used to inspect the verifier and its exception tables.

## 7. Key Takeaways

The main technique is **exception-based control-flow obfuscation**.

When reversing a similar APK:

1. Do not assume every `throw` represents an error.
2. Inspect the DEX `try`/`catch` tables.
3. Treat matching `catch` blocks as possible basic-block destinations.
4. Reconstruct the control-flow graph rather than following the code sequentially.
5. Separate genuine success paths from failure strings and decoys.
6. Check the challenge's required flag wrapper before submitting an embedded value.

The essential transformation is:

```text
Normal:

if (condition)
    A;
else
    B;


Obfuscated:

try {
    throw BranchAExc;
} catch (BranchAExc e) {
    A;
} catch (BranchBExc e) {
    B;
}
```

Once the exception handlers are treated as control-flow edges, the verifier becomes much easier to follow.

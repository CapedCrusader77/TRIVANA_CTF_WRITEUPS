# U --- Exception Maze

**Category:** Reverse Engineering / Android\
**Difficulty:** Medium\
**Points:** 150

## Overview

This Android reversing challenge hides its decision-making logic behind
Java/DEX exceptions. Instead of relying on the usual `if`, `else`, and
branch instructions, the verifier uses a custom family of exceptions
named `BranchExceptions`.

The objective is to identify which exceptions correspond to valid
transitions, discard the deliberately misleading paths, and reach the
value used by the successful verification route.

## 1. Start with the APK

The challenge provides:

``` text
chal08.apk
```

A quick first pass can be done with:

``` bash
file chal08.apk
unzip -l chal08.apk | head
aapt dump badging chal08.apk
```

Then decode the application:

``` bash
apktool d chal08.apk -o chal08_dec
```

This exposes the application's smali representation.

The custom exception classes are a useful starting point:

``` bash
grep -R "Branch" chal08_dec/smali* 2>/dev/null
```

From there, focus on the verifier and the classes implementing
`BranchExceptions`.

## 2. Why the verifier looks strange

A normal verifier might look conceptually like this:

``` java
if (valid) {
    continueChecking();
} else {
    reject();
}
```

Here, the same decision is represented through exception handling. An
exception is deliberately thrown and the type of that exception
determines which handler receives execution.

A simplified model is:

``` text
                    validation step
                         |
                  evaluate condition
                    /           \
                   /             \
          BranchAExc          BranchBExc
               |                   |
               v                   v
        next validation         reject
               |
          next condition
            /       \
           /         \
   success path    BranchDExc
                       |
                       v
                     reject
```

The important point is that `throw` is being used as an obfuscated jump
mechanism. Treating every exception as an error leads to the wrong
interpretation of the program.

## 3. Rebuild the hidden flow

The exception table and the surrounding handlers need to be read
together.

One route ends in:

``` text
BranchBExc
    |
    v
invalid_branch_b
```

That is a dead-end rejection path.

Another route is:

``` text
BranchAExc
    |
    v
next validation
```

This is the useful transition. The exception is effectively acting like
a conditional jump to the next verification stage.

The same pattern appears later. For example:

``` text
BranchDExc
    |
    v
invalid_branch_d
```

is another rejection route.

By following the handlers rather than simply reading the smali
sequentially, the verifier's real control-flow graph becomes much
clearer.

## 4. Locate the successful value

Once the valid handler chain is followed to completion, the embedded
success string can be recovered:

``` text
STUSEC{3xc3p710n_cf6_sm4l1_6r4ph_r3c0ns7ruc710n_6194}
```

The APK itself stores the value with the `STUSEC` wrapper. The challenge
submission format, however, requires:

``` text
TRIVARNA{...}
```

Therefore the payload remains unchanged while only the outer wrapper is
converted to the required format.

## 5. Final flag

``` text
TRIVARNA{3xc3p710n_cf6_sm4l1_6r4ph_r3c0ns7ruc710n_6194}
```

## 6. Reproduction commands

A compact workflow is:

``` bash
file chal08.apk
aapt dump badging chal08.apk
apktool d chal08.apk -o chal08_dec
grep -R "Branch" chal08_dec/smali* 2>/dev/null
grep -R "invalid_branch" chal08_dec/smali* 2>/dev/null
grep -R "STUSEC" chal08_dec/smali* 2>/dev/null
```

For a higher-level view, JADX can be useful for recovering Java-like
structure, while Ghidra or baksmali can help when the DEX/smali
representation needs closer inspection.

## 7. Lessons from the challenge

-   An exception in obfuscated code is not automatically an error
    condition.
-   DEX `try`/`catch` metadata can reveal hidden control-flow edges.
-   Custom exception subclasses may encode different branches of a
    verifier.
-   Failure strings are useful landmarks, but they should not be
    confused with the successful route.
-   The final submission format may differ from the wrapper used by an
    embedded string.

### Control-flow transformation

The intended idea can be reduced to:

``` text
Ordinary program:

        condition
        /      \
       A        B


Obfuscated verifier:

        throw X
          |
      catch X
       /    \
      A      B
```

The trick is not the exception itself; it is the fact that exception
dispatch has been turned into the branch mechanism. Once those handlers
are mapped as edges in a control-flow graph, the verifier stops looking
nearly as complicated.

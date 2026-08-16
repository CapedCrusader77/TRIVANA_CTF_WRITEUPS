# U - My First IOT

## Challenge

**Difficulty:** Easy  
**Points:** 150

My First IOT project.

## Objective

Recover the hidden flag from the supplied `my-first-iot.tgz`.

## Analysis

The extracted files looked like:

- `config.txt`
- `debug.log`
- `README.txt`

However, each file was actually a ZIP archive with a misleading extension. Each archive extracted to a plain-text file with the same name.

This created a nested archive / Matryoshka-style extraction chain.

`README.txt` pointed toward `debug.log` as the important artifact.

## Debug Log Analysis

The final `debug.log` contained a fake ESP32-style boot log mixed with normal sensor telemetry.

Five specially marked fragments were embedded in the log using the form:

```text
frag[n/5]
```

The fragments were Base64 encoded.

The correct procedure was:

1. Find the five `frag[n/5]` entries.
2. Order them by their fragment number.
3. Concatenate their Base64 values.
4. Base64-decode the resulting string.

This produced:

```text
UNI6CTF{d3bug_l0gs_never_lie}
```

The `UNI6CTF` wrapper was a leftover/template clue, while this event requires the `TRIVARNA{...}` wrapper.

## Final Flag

```text
TRIVARNA{d3bug_l0gs_never_lie}
```

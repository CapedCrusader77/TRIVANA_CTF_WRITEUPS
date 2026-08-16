# Dynamic DEX Loading

## Challenge
**File:** `chal02.apk`  
**Difficulty:** Medium  
**Points:** 150

## Solution
The APK contains an `IntegrityGate` that derives an AES-256 key through HKDF-SHA256 from:

- SHA-256 of the APK signing certificate
- CRC32 of `classes.dex`

HKDF parameters:

```text
Salt: STUSEC_SALT_2026
Info: DEX_DEC_KEY
```

The resulting key decrypts:

```text
assets/stage2.dex.enc
```

using AES-GCM. The decrypted payload is a valid DEX and can be analyzed offline, so the application's runtime anti-debug/anti-hook checks are unnecessary for solving the challenge.

The stage-2 DEX is loaded dynamically with `InMemoryDexClassLoader`.

`EntryPoint.run()` contains the final flag logic. `AntiDebug.check()` uses a native ptrace-style check and `AntiHook.check()` scans `/proc/self/maps` for Frida/Xposed indicators, but neither affects the static decryption route.

`FlagTransform.deriveFlag()` ultimately uses a hardcoded 50-byte `SEED`. After removing a 4-byte `MAGIC` prefix, it XORs the remaining data with the repeating key:

```text
STUSEC_ROLL_2026
```

Recovered flag body:

```text
dYn4m1c_d3x_l04d1ng_4nd_4r7_4n71_d3bu6_byp4ss_7382
```

The app internally uses `STUSEC{...}`, but the event requires `TRIVARNA{...}`.

## Final Flag

```text
TRIVARNA{dYn4m1c_d3x_l04d1ng_4nd_4r7_4n71_d3bu6_byp4ss_7382}
```

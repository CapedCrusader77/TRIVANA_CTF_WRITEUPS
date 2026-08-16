# Time/Sensor/Environment-Gated Flag

**Category:** Mobile / Reverse Engineering
**Difficulty:** Medium
**Points:** 200
**Solves:** 1

---

## Challenge Description

> This application uses a multi-factor environmental validation chain in native code (`libcheck.so`) before releasing the flag:
>
> 1. **Accelerometer Noise Floor Sampling:** Samples accelerometer sensor readings over 2–3 seconds and calculates variance. Stock emulators producing zero noise floor or synthetic flatlines fail.
> 2. **Device Fingerprint Whitelisting:** Checks `Build.FINGERPRINT` against a hash whitelist table.
> 3. **Rolling Boot-Time TOTP Window:** Validates a rolling time value calculated from `SystemClock.elapsedRealtime()` with a 3-step (90-second) tolerance window.

---

## Solution

### Step 1 — Unpack the APK

An APK is just a ZIP archive. Unpack it to access the internal files:

```bash
unzip chal07.apk -d apk_unpacked
ls apk_unpacked/
# AndroidManifest.xml  META-INF  classes.dex  kotlin  lib  resources.arsc
```

### Step 2 — Locate the Native Library

The challenge description points directly to `libcheck.so`. Two ABI variants are present:

```
apk_unpacked/lib/arm64-v8a/libcheck.so
apk_unpacked/lib/x86_64/libcheck.so
```

We use the `x86_64` build since it can be disassembled with standard tools.

### Step 3 — Run `strings` on the Library

Before reaching for a disassembler, always start with `strings`. The flag is stored **in plaintext** in the binary's read-only data section:

```bash
strings apk_unpacked/lib/x86_64/libcheck.so
```

Relevant output:

```
Java_com_ctf_chal07_EnvCheck_nativeDeriveFlag
512184b2b799bfbf3b33b134dd1320bca856a3d6ebdc9187c527d83cda61eac2
40669084df34c6ad268d25004b9bbcdf8f872444c2aa7faa474a24bad7bc75b9
6af850e1d32086c4d86f0b18bf4dc0a7cdb73e6c3d977877451c715cc7c72ebd
STUSEC{3nv_6473d_s3ns0r_f1n63rpR1n7_707p_c0mb1n3_3910}
```

The flag is immediately visible. The three SHA-256 hashes are the device fingerprint whitelist entries referenced in the challenge description.

### Step 4 — Verify via Disassembly

Disassembling `nativeDeriveFlag` confirms the flag is never encrypted — it is simply returned conditionally:

```bash
objdump -d apk_unpacked/lib/x86_64/libcheck.so
```

Key logic reconstructed from the disassembly:

```c
jstring Java_com_ctf_chal07_EnvCheck_nativeDeriveFlag(
    JNIEnv *env, jobject thiz,
    jstring fingerprint,       // Build.FINGERPRINT
    jdouble accel_variance,    // sensor noise floor
    jlong totp_step,           // elapsedRealtime() / 30000
    jlong totp_min,
    jlong totp_max
) {
    // Check 1: Device fingerprint against SHA-256 whitelist
    const char *fp = (*env)->GetStringUTFChars(env, fingerprint, 0);
    bool fp_ok = strcmp(fp, "512184b2b...") == 0 ||
                 strcmp(fp, "40669084d...") == 0 ||
                 strcmp(fp, "6af850e1d...") == 0;
    (*env)->ReleaseStringUTFChars(env, fingerprint, fp);

    // Check 2: Accelerometer variance must be in range [0.001, 0.2]
    bool accel_ok = accel_variance >= 0.001 && accel_variance <= 0.2;

    // Check 3: TOTP window — elapsedRealtime must be > 90000ms (boot time sanity)
    //           and within a valid rolling 3-step window
    bool totp_ok = totp_step >= 0x15F90 &&   // 90000 decimal
                   totp_step <  totp_max &&
                   totp_max  <  totp_min;

    if (fp_ok && accel_ok && totp_ok) {
        // Flag returned only if ALL checks pass
        return (*env)->NewStringUTF(env, "STUSEC{3nv_6473d_s3ns0r_f1n63rpR1n7_707p_c0mb1n3_3910}");
    }
    return NULL;
}
```

The flag string at offset `0x5e3` in `.rodata` is passed directly to `NewStringUTF` — no decryption, no XOR, no runtime derivation. The environmental checks are purely gating logic.

### Step 5 — Recover the Flag

The inner flag content is wrapped in the event format:

```
TRIVARNA{3nv_6473d_s3ns0r_f1n63rpR1n7_707p_c0mb1n3_3910}
```

---

## Key Takeaway

The challenge description is designed to intimidate — accelerometer sampling, fingerprint hashing, and TOTP windows sound complex. In practice, **the flag was never obfuscated**; it lives in `.rodata` as a plaintext string. All three checks are runtime *gates*, not cryptographic protections.

**Lesson:** Always run `strings` before spinning up an emulator or writing a bypass. Binary analysis starting from the simplest tool often wins.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Unpack the APK |
| `strings` | Extract plaintext strings from the `.so` |
| `objdump -d` | Disassemble and verify control flow |
| Python `bytes.find()` | Confirm offsets and surrounding context |

---

## Flag

```
TRIVARNA{3nv_6473d_s3ns0r_f1n63rpR1n7_707p_c0mb1n3_3910}
```

# VaultX Wallet — Writeup

**Category:** Mobile / Reverse Engineering
**Difficulty:** Medium
**Points:** 350
**Flag format:** `TRIVARNA{word1_word2_..._word12}`

---

## Challenge Description

> A digital asset wallet developed for field officers claims to protect
> recovery phrases using military-grade encryption. Investigation reveals
> that convenience features accidentally expose the wallet secret through
> multiple storage locations.
>
> **Goal:** Recover the wallet's 12-word recovery (seed) phrase.

Provided file: `player_files.zip` → contains a single file, **`VaultX.apk`**.

---

## 1. Triage

```
$ file VaultX.apk
VaultX.apk: Android package (APK), with gradle app-metadata.properties,
with APK Signing Block
```

This is a standard Android application (Kotlin, package `com.vaultx.wallet`),
built with several dex files (`classes.dex` … `classes6.dex`), AndroidX /
Material / Room dependencies, and no native libraries or assets.

Decompiled with **jadx 1.5.6**:

```
jadx -d jadx_out VaultX.apk
```

App-specific code lives entirely under `sources/com/vaultx/wallet/`:

```
MainActivity.java
DebugActivity.java
VaultXApp.java
manager/WalletManager.java
crypto/VaultCrypto.java
db/AppDatabase.java
db/Transaction.java
db/TransactionDao.java
```

---

## 2. Finding the Seed Storage — `WalletManager`

```kotlin
private val SEED_WORDS = arrayOf(
    "ocean", "maple", "helmet", "alpha", "bravo", "delta",
    "echo", "foxtrot", "golf", "hotel", "india", "juliet"
)
private val SEED_PHRASE by lazy { SEED_WORDS.joinToString(" ") }

fun initializeWallet() {
    if (!prefs.contains(ENCRYPTED_SEED_KEY)) {
        val encrypted = VaultCrypto.encrypt(SEED_PHRASE)
        prefs.edit().putString(ENCRYPTED_SEED_KEY, encrypted).apply()
    }
}

fun getEncryptedSeed(): String? = prefs.getString(ENCRYPTED_SEED_KEY, null)

fun getDecryptedSeed(): String? =
    try { VaultCrypto.decrypt(getEncryptedSeed()!!) } catch (e: Exception) { null }
```

On first launch, `VaultXApp.onCreate()` calls `initializeWallet()`, which
encrypts the hardcoded `SEED_WORDS` and writes it to `SharedPreferences`
under the key `wallet_prefs / encrypted_seed`. This is the "military-grade
encryption" the challenge blurb references — in practice it's just a
hardcoded value encrypted with a hardcoded key (see next section).

---

## 3. Breaking the Encryption — `VaultCrypto`

```kotlin
object VaultCrypto {
    private const val AES_KEY = "VAULTX2026AES!"

    private val keySpec by lazy {
        SecretKeySpec(AES_KEY.toByteArray(Charsets.UTF_8).copyOf(16), "AES")
    }

    fun encrypt(plaintext: String): String {
        val cipher = Cipher.getInstance("AES/ECB/PKCS5Padding")
        cipher.init(Cipher.ENCRYPT_MODE, keySpec)
        return Base64.encodeToString(cipher.doFinal(plaintext.toByteArray()), 0)
    }

    fun decrypt(ciphertext: String): String {
        val cipher = Cipher.getInstance("AES/ECB/PKCS5Padding")
        cipher.init(Cipher.DECRYPT_MODE, keySpec)
        return String(cipher.doFinal(Base64.decode(ciphertext.trim(), 0)))
    }

    fun isValidKey(candidate: String?) = candidate == AES_KEY
}
```

**Vulnerability #1 — Hardcoded AES key in bytecode.**
`AES/ECB` with a static, embedded key means anyone with the APK can decrypt
`encrypted_seed` offline, no device or backdoor required.

`VaultCrypto` also contains a decoy int-array "fragment hint" (a separate
sub-flag for this vuln, unrelated to the wallet seed): decoding the array
gives `FLAG1{hardcoded_crypto_keys_in_bytecode_flag_one}`.

---

## 4. The "Convenience Feature" — `DebugActivity`

```kotlin
class DebugActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_debug)

        val providedKey = intent.getStringExtra("debug_key")
        val seed = if (VaultCrypto.isValidKey(providedKey)) {
            (application as VaultXApp).walletManager.decryptedSeed
        } else null

        seedDisplay.text = seed ?: "Invalid debug key"

        copyButton.setOnClickListener {
            if (seed != null) {
                clipboard.setPrimaryClip(ClipData.newPlainText("Recovery Phrase", seed))
                // reveals a second decoy fragment on the UI + clipboard
            }
        }
    }
}
```

`DebugActivity` is `android:exported="true"` in the manifest, meaning **any
app on the device can launch it directly** and supply `debug_key` as an
intent extra:

```
adb shell am start -n com.vaultx.wallet/.DebugActivity \
    --es debug_key "VAULTX2026AES!"
```

Since the key is the same hardcoded `AES_KEY` used for encryption, this
"debug panel" trivially decrypts and displays the wallet seed in the UI —
and the **Copy** button pushes it straight to the **system clipboard**,
which is readable by other apps on many Android versions. This is
vulnerability #2/#3: **debug backdoor + clipboard leakage**.

The copy button also reveals a second decoy fragment, extracted from a raw
`int[]` in the bytecode (values must be read from raw dex instructions —
jadx mis-decompiles some of the literals as unrelated Android SDK constant
names). Decoded correctly:

```
FLAG3{clipboard_data_leakage_detected}
```

---

## 5. Additional exposure surface (not required to solve, noted for completeness)

- `android:allowBackup="true"` in the manifest — the app's `SharedPreferences`
  (containing `encrypted_seed`) is extractable via `adb backup` without root,
  a fourth "storage location" the encrypted seed lives in.
- `android:debuggable="true"` — trivially attachable with a debugger/Frida
  to call `WalletManager.getDecryptedSeed()` directly at runtime.

These are the "multiple storage locations" referenced in the challenge
description: SharedPreferences (encrypted), device backup, the debug
activity UI, and the clipboard.

---

## 6. Recovering the Seed Phrase

Putting it together — hardcoded key → decrypts hardcoded ciphertext →
recovered via the debug backdoor:

```
Recovery phrase:
ocean maple helmet alpha bravo delta echo foxtrot golf hotel india juliet
```

**Note:** roughly half of these words (`bravo`, `delta`, `foxtrot`, `golf`,
`india`, `juliet`) are NATO phonetic-alphabet words, not part of the
official BIP39 English wordlist — so this phrase will *not* pass a strict
BIP39 checksum validator. It is nonetheless the only seed value present
anywhere in the app (confirmed via exhaustive review of every class,
layout, and string resource), and it's exactly the value the app's own
debug feature decrypts and displays as "the recovery phrase." The
checksum mismatch appears to be intentional flavor — a reminder that a
"military-grade" wallet that fails basic BIP39 validation was never
secure to begin with.

---

## Final Flag

```
TRIVARNA{ocean_maple_helmet_alpha_bravo_delta_echo_foxtrot_golf_hotel_india_juliet}
```

---

## Tools Used

- `jadx` 1.5.6 — dex → Java decompilation
- `androguard` — raw bytecode/instruction inspection (to recover true int
  literals where jadx substituted unrelated SDK constant names)
- `python3` — array decoding, BIP39 wordlist cross-check

## Root Causes (Summary)

| # | Vulnerability | Location |
|---|---|---|
| 1 | Hardcoded AES key, ECB mode | `VaultCrypto.kt` |
| 2 | Exported debug activity exposing decrypted secret | `DebugActivity.kt` + manifest |
| 3 | Secret copied to system clipboard | `DebugActivity.kt` |
| 4 | `allowBackup=true` exposes encrypted prefs via `adb backup` | `AndroidManifest.xml` |
| 5 | `debuggable=true` in release-adjacent build | `AndroidManifest.xml` |

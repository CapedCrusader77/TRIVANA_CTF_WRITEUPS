# Saffron Echoes in Old Delhi — Writeup

**Category:** Forensics / Steganography / Crypto
**Difficulty:** Medium · 300 points · 2 solves
**Flag:** `TRIVARNA{AR3_Y0U_U51NG_C1AU4E_F04_S01V1NG_TH15_CH@LL3NG3?}`

---

## Challenge

> An investigator has recovered a seized Linux disk image from a courier's workstation in Old Delhi. The drive appears normal, but operators suspect the final transfer records — hidden layers of steganography and encryption — still linger within its sectors, waiting to echo the secrets of that independence night. Your mission: recover the hidden payloads, peel back the layered obfuscation, and decrypt the final message to reveal the truth buried beneath.

**Artifact:** `saffron_echoes.img` (268,435,456 bytes / 256 MB)

| Hash | Value |
|---|---|
| MD5 | `ec50e963723045fd3b2722f3a4a9df2e` |
| SHA-256 | `ad8037a7d9e08f83c59fbacfd21aaf36e0f58970897adb9618d2f33376e68db3` |

---

## TL;DR

1. The `.img` is a bare **ext4 filesystem** (no partition table) — mount it or carve it with `debugfs`.
2. The extracted home directory is a staged crime scene: chat logs, mail, notes and a `challenge_runtime.json` that leak two sets of credentials.
3. Four **BMP** scans carry **steghide** payloads. Three open with the *decoy* passphrase; exactly one opens only with the *final* passphrase, which you must reconstruct from the notes + word list.
4. That final payload is **AES-256-CBC** with a **PBKDF2-HMAC-SHA256 (10,000 iterations)** derived key. Decrypt it to get the flag.

---

## Step 1 — Identify the image

```bash
$ file saffron_echoes.img
saffron_echoes.img: Linux rev 1.0 ext4 filesystem data,
  UUID=f83285e2-bb1a-4a34-bdec-b901cf985c4e, volume name "KASHI_LEDGER"
  (extents) (64bit) (large files) (huge files)
```

There is no MBR/GPT — the filesystem starts at offset 0. That means `mmls`/`fdisk` are useless here and you go straight at it.

With root you could `mount -o loop,ro`. Without root (or on a forensics box where you'd rather not mount), `debugfs` from `e2fsprogs` does everything:

```bash
$ debugfs -R "ls -l /" saffron_echoes.img
      2   40777 (2)   1000  1000   4096 26-Mar-2026 10:21 .
     11   40700 (2)      0     0  16384 26-Mar-2026 10:21 lost+found
     12   40777 (2)   1000  1000   4096 26-Mar-2026 10:21 deleted_mail_pool
  32769   40777 (2)      0     0   4096 26-Mar-2026 10:21 etc
  32770   40777 (2)   1000  1000   4096 26-Mar-2026 10:21 home
     13  100777 (1)   1000  1000    631 26-Mar-2026 10:21 challenge_runtime.json
```

Dump the whole tree:

```bash
$ mkdir ext && debugfs -R "rdump / ./ext" saffron_echoes.img
```

Worth doing early, and it comes back empty here — no deleted-inode rabbit hole:

```bash
$ debugfs -R "lsdel" saffron_echoes.img
0 deleted inodes found.
```

### Recovered tree

```
/
├── challenge_runtime.json
├── etc/{hostname,issue}
├── deleted_mail_pool/{philosophy_01.eml,philosophy_02.eml}
└── home/pandit_ved/
    ├── Documents/ritual_words.txt
    ├── Notes/{ritual_index_notes.md,restoration_log.txt}
    ├── Browser/history.tsv
    ├── Maildir/{new,cur}/...
    ├── chatlogs/{ward-chat-2026-03-11.log,ward-chat-2026-03-14.log}
    ├── Pictures/ward_scans/*.bmp          <-- the payloads
    ├── .archive_payloads/*.{txt,enc}
    └── .cache/.glyph_index.bin            <-- red herring
```

---

## Step 2 — Loot the workstation for credentials

### `challenge_runtime.json`

This is the shortcut — it hands you both credential sets outright:

```json
{
  "stage2_phrase": "ghat-manjari-copper-owl",
  "stage1": {
    "steg_passphrase": "trishul-lantern-braid",
    "aes": {
      "salt": "9a31f4b20d17c8ef",
      "key": "amber-ledger-lintel",
      "iv": "43a65f90d4bbcb17850a79c2e36d1f4a"
    }
  },
  "stage2": {
    "steg_words": ["ghat", "manjari", "copper", "owl"],
    "aes": {
      "salt": "d83f0a1e5bc94762",
      "key": "river-ink-oblation",
      "iv": "7f01ea25d6c59f4b38e4d0b451ccae12"
    }
  }
}
```

But the *intended* path reconstructs stage 2 from the narrative, so it's worth walking that too — it's the actual puzzle.

### `chatlogs/ward-chat-2026-03-11.log` — stage 1 only

```
[2026-03-11 22:06] ved: Keep this offline. First lock phrase is: trishul-lantern-braid
[2026-03-11 22:08] tara: only for the decoy capsules?
[2026-03-11 22:09] ved: yes, and decrypt with these if extracted:
[2026-03-11 22:10] ved: AES-256-CBC salt=9a31f4b20d17c8ef
[2026-03-11 22:10] ved: AES-256-CBC key=amber-ledger-lintel
[2026-03-11 22:11] ved: AES-256-CBC iv=43a65f90d4bbcb17850a79c2e36d1f4a
[2026-03-11 22:12] tara: understood, those are for false ledgers and ash manifests.
```

Note the explicit warning: these are **decoy** credentials.

### `chatlogs/ward-chat-2026-03-14.log` — the constraint

```
[2026-03-14 00:18] tara: did you reuse trishul-lantern-braid on the final capsule?
[2026-03-14 00:19] ved: no. final one follows my passphrase doctrine from mail,
                        and starts with north alcove pair.
[2026-03-14 00:21] tara: then brute-force will still be feasible if someone gets your word list.
[2026-03-14 00:22] ved: only two words unknown if they read notes carefully.
```

### `deleted_mail_pool/philosophy_01.eml` — the doctrine

```
Subject: passphrase doctrine draft

Reminder for all personal lock phrases:
1) Always compose exactly four words.
2) First two words are location labels.
3) Last two words must be animals or objects from ritual_words list.
4) Join with hyphen; no numerals.
```

### `Notes/ritual_index_notes.md` — the prefix

```markdown
# Ritual Ledger Scratch Notes

- The second lock phrase follows the ward naming rule from the old roster.
- Prefix is fixed from the north alcove labels: ghat manjari ____ ____
- Last two words are always from the curated ritual_words list.
- Never reuse the first lock phrase for final capsule.
- Confirmed: hidden ledger capsule is in one scan that does not open with standard phrase.
```

### `Documents/ritual_words.txt` — the search space

```
ghat  manjari  copper  owl    lotus   mirror  vellum  ashes
lantern  chisel  saffron  knot  census  reed  trunk  prayer
```

**Putting it together:** the final passphrase is `ghat-manjari-<w3>-<w4>` where `w3, w4` come from a 16-word list. That's at most `16 × 16 = 256` candidates — trivially brute-forceable even without the JSON, which is exactly what tara predicted in the chat log.

The answer is `ghat-manjari-copper-owl`.

### `deleted_mail_pool/philosophy_02.eml` — the misdirection

```
Do not store the second-stage AES tuple near the payload.
Move it where only block-level examiners will look.
```

This baits you toward slack space / unallocated block analysis. The tuple is actually just in `challenge_runtime.json`. Similarly, `Browser/history.tsv` contains a link to a *"Balanced Ternary Primer"*, and `.cache/.glyph_index.bin` is 4097 bytes of high-entropy noise. **Neither is used.** They exist to burn your time.

---

## Step 3 — steghide on the four BMPs

```bash
$ file Pictures/ward_scans/*.bmp
scan_ghat_registry.bmp:  PC bitmap, Windows 3.x format, 320 x 240 x 24, ...
scan_lacquer_margin.bmp: PC bitmap, Windows 3.x format, 320 x 240 x 24, ...
scan_midnight_index.bmp: PC bitmap, Windows 3.x format, 320 x 240 x 24, ...
scan_river_ledger.bmp:   PC bitmap, Windows 3.x format, 320 x 240 x 24, ...
```

Identical geometry, identical file size (230,454 bytes), no trailing data past the pixel array. 24-bit BMP is a classic **steghide** carrier.

Spray both passphrases across all four:

```bash
for f in *.bmp; do
  for p in trishul-lantern-braid ghat-manjari-copper-owl; do
    steghide extract -sf "$f" -p "$p" -xf "out_${f%.bmp}_$p.bin" -f 2>&1 \
      | sed "s|^|[$f / $p] |"
  done
done
```

Result:

| BMP | `trishul-lantern-braid` | `ghat-manjari-copper-owl` | Size |
|---|---|---|---|
| `scan_ghat_registry.bmp` | ✅ | ❌ | 96 B |
| `scan_lacquer_margin.bmp` | ✅ | ❌ | 80 B |
| `scan_river_ledger.bmp` | ✅ | ❌ | 80 B |
| **`scan_midnight_index.bmp`** | ❌ | ✅ | **208 B** |

This matches the note exactly: *"the hidden ledger capsule is in one scan that does not open with the standard phrase."*

**Brute-force fallback** (if you never found `challenge_runtime.json`):

```bash
words=$(cat Documents/ritual_words.txt)
for a in $words; do for b in $words; do
  steghide extract -sf scan_midnight_index.bmp -p "ghat-manjari-$a-$b" \
    -xf capsule.bin -f >/dev/null 2>&1 && echo "HIT: ghat-manjari-$a-$b"
done; done
# HIT: ghat-manjari-copper-owl
```

256 attempts, sub-second. The challenge is designed so this always works.

---

## Step 4 — Break the AES layer

The extracted 208-byte blob is raw ciphertext — **no `Salted__` OpenSSL header**, so the salt is not embedded; it's the one from the JSON. The IV is given explicitly, which rules out OpenSSL's `EVP_BytesToKey` (that derives IV *and* key together).

That leaves the question of the KDF. Salt + separate IV + a passphrase-looking key strongly implies **PBKDF2**. Iteration count is the only unknown — test the common ones:

```python
import hashlib
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

data = open('out_scan_midnight_index_ghat-manjari-copper-owl.bin', 'rb').read()
salt = bytes.fromhex('d83f0a1e5bc94762')
iv   = bytes.fromhex('7f01ea25d6c59f4b38e4d0b451ccae12')
pw   = b'river-ink-oblation'

candidates = {
    'pbkdf2-1k':     hashlib.pbkdf2_hmac('sha256', pw, salt, 1000,   32),
    'pbkdf2-10k':    hashlib.pbkdf2_hmac('sha256', pw, salt, 10000,  32),
    'pbkdf2-100k':   hashlib.pbkdf2_hmac('sha256', pw, salt, 100000, 32),
    'sha256(pw)':    hashlib.sha256(pw).digest(),
    'sha256(s+pw)':  hashlib.sha256(salt + pw).digest(),
    'sha256(pw+s)':  hashlib.sha256(pw + salt).digest(),
    'scrypt':        hashlib.scrypt(pw, salt=salt, n=16384, r=8, p=1, dklen=32),
}

for name, key in candidates.items():
    dec = Cipher(algorithms.AES(key), modes.CBC(iv)).decryptor()
    print(name, (dec.update(data) + dec.finalize())[:60])
```

`pbkdf2-10k` is the hit. Full decrypt with PKCS#7 unpadding:

```python
key = hashlib.pbkdf2_hmac('sha256', b'river-ink-oblation',
                          bytes.fromhex('d83f0a1e5bc94762'), 10000, 32)
dec = Cipher(algorithms.AES(key),
             modes.CBC(bytes.fromhex('7f01ea25d6c59f4b38e4d0b451ccae12'))).decryptor()
pt = dec.update(data) + dec.finalize()
print(pt[:-pt[-1]].decode())
```

Equivalent one-liner:

```bash
openssl enc -d -aes-256-cbc -in capsule.bin \
  -K $(python3 -c "import hashlib;print(hashlib.pbkdf2_hmac('sha256',b'river-ink-oblation',bytes.fromhex('d83f0a1e5bc94762'),10000,32).hex())") \
  -iv 7f01ea25d6c59f4b38e4d0b451ccae12
```

### Plaintext

```
Hidden Ledger Capsule
====================

Reconstructed transfer note, sealed for tribunal audit.
Flag: UNI6CTF{AR3_Y0U_U51NG_C1AU4E_F04_S01V1NG_TH15_CH@LL3NG3?}

Custodian phrase seed: ghat manjari
```

> **Note on the wrapper:** the capsule embeds a `UNI6CTF{...}` prefix (an artifact of the challenge's origin), while this event's submission format is `TRIVARNA{...}`. The inner content is unchanged — swap the wrapper when submitting.

---

## Flag

```
TRIVARNA{AR3_Y0U_U51NG_C1AU4E_F04_S01V1NG_TH15_CH@LL3NG3?}
```

---

## Full solve script

```bash
#!/usr/bin/env bash
set -e

# 1. Carve the ext4 filesystem
mkdir -p ext && debugfs -R "rdump / ./ext" saffron_echoes.img 2>/dev/null
cd ext/home/pandit_ved

# 2. Recover the capsule (brute-force the last two words)
WORDS=$(cat Documents/ritual_words.txt)
cd Pictures/ward_scans
for a in $WORDS; do for b in $WORDS; do
  if steghide extract -sf scan_midnight_index.bmp \
       -p "ghat-manjari-$a-$b" -xf /tmp/capsule.bin -f >/dev/null 2>&1; then
    echo "[+] passphrase: ghat-manjari-$a-$b"; break 2
  fi
done; done

# 3. Decrypt
python3 - <<'PY'
import hashlib
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
d   = open('/tmp/capsule.bin','rb').read()
key = hashlib.pbkdf2_hmac('sha256', b'river-ink-oblation',
                          bytes.fromhex('d83f0a1e5bc94762'), 10000, 32)
iv  = bytes.fromhex('7f01ea25d6c59f4b38e4d0b451ccae12')
c   = Cipher(algorithms.AES(key), modes.CBC(iv)).decryptor()
p   = c.update(d) + c.finalize()
print(p[:-p[-1]].decode())
PY
```

---

## Decoys and dead ends

Worth cataloguing, since most of the challenge's difficulty is in *not* chasing these:

| Artifact | What it looks like | What it is |
|---|---|---|
| `.cache/.glyph_index.bin` | 4097 bytes, high entropy, hidden dotfile | Pure CSPRNG noise. No XOR key, no structure, no strings. |
| `Browser/history.tsv` → "Balanced Ternary Primer" | Suggests a ternary encoding stage | Never used anywhere in the chain. |
| `philosophy_02.eml` "block-level examiners" | Points at slack/unallocated space | The AES tuple is in a plain JSON file. |
| `.archive_payloads/decoy_ledger_{a,b,c}.{txt,enc}` | Pre-extracted payloads | The three stage-1 decoys, already solved for you. |
| `Notes/restoration_log.txt` | Timestamped activity log | Pure flavour text. |
| `lost+found` | Standard deleted-file hunting ground | Empty. `lsdel` reports 0 deleted inodes. |
| `deleted_mail_pool/` | "deleted" implies recovery work | Files are fully intact and readable. |

Checks that came back clean, so you can skip them: raw `grep` for the flag wrapper across all 256 MB, full ASCII strings sweep, archive-signature carving (zip/gzip/png/7z/rar/xz/bz2/tar), LSB extraction on all four BMPs, trailing-bytes-past-pixel-array inspection, and single-byte XOR of `.glyph_index.bin`.

---

## Takeaways

- **`file` before anything else.** A `.img` with no partition table is common in CTF forensics — going straight to `debugfs`/`mount` beats fighting `mmls`.
- **`debugfs -R "rdump /"` is the no-root workhorse.** Full recursive extraction without loop-mount privileges, plus `lsdel` to check deleted inodes in one shot.
- **Read the narrative for the constraint, not just the credentials.** The notes reduce the passphrase space from unbounded to 256 candidates; the chat log even tells you the brute-force is feasible.
- **Test KDFs systematically.** An explicit IV alongside a salt rules out OpenSSL's `EVP_BytesToKey` and points at PBKDF2. Scripting six or seven candidates in one pass is faster than reasoning about which one the author picked.
- **Budget time for decoys.** Roughly half the artifacts here exist purely to consume effort. Anything with no thread connecting it to a credential or a payload is probably scenery.

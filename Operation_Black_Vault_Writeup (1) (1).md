# Operation Black Vault — Writeup

**Category:** Reverse Engineering / Forensics
**Difficulty:** Medium (500 pts)
**Target:** `BlackVault` — a .NET 8 / WPF desktop "evidence management" application

**Flag:** `TRIVARNA{ST4T1C_K3YS_PLUG1N_D3S3R1AL1Z3_ENCRYPT3D}`

---

## 1. Challenge Overview

Black Vault is a shipped .NET 8 WPF desktop application (`BlackVault.exe` / `BlackVault.dll`) bundled with:

- A plugin (`Plugins/DemoPlugin.dll`)
- A config file (`config.json`)
- A sample encrypted case package (`sample.casepkg`)
- Standard .NET/SQLite runtime dependencies

The prompt states four flag fragments are hidden across the app's **code, plugins, database, and configuration systems**, and that they must be assembled into a master access key.

---

## 2. Tooling / Setup

Static analysis was done entirely offline against the shipped binaries — no live app execution was required.

```bash
pip install dnfile dncil pycryptodome --break-system-packages
apt-get install -y mono-utils   # monodis, for a first-pass IL dump
```

- **`dnfile`** — parses .NET PE metadata tables (`TypeDef`, `Field`, `FieldRVA`, `Constant`, the user-strings heap, etc.) directly, without needing to resolve external assembly references.
- **`dncil`** — disassembles individual method bodies into CIL instructions from raw RVA offsets.
- **`monodis`** was tried first but segfaults partway through the assembly (it tries to resolve `System.Runtime` types it can't load in this sandbox). `dnfile`/`dncil` sidestep this entirely by reading metadata directly, which is what unblocked full analysis.

---

## 3. Recon — Enumerating the Attack Surface

Listing every `MethodDef` and every entry in the user-strings (`#US`) heap gave a full map of the app in a few minutes:

```python
import dnfile
pe = dnfile.dnPE("BlackVault.dll")
for m in pe.net.mdtables.MethodDef.rows:
    print(m.Name.value)
```

Key classes discovered, **in declaration order** (this order turns out to matter — see §8):

```
TypeDef #3  ArchiveImporter
TypeDef #4  CaseImporter
TypeDef #5  CryptoManager        ← static AES key lives here
TypeDef #6  ImportManager
TypeDef #7  MainWindow
TypeDef #8  PluginLoader         ← discovers/loads Plugins/*.dll
...
TypeDef #20 DiagnosticAction     ← dead code, never instantiated
...
TypeDef #27 EvidenceDatabase     ← SQLite wrapper (Cases, Evidence, Flags, Logs)
```

Manually walking the user-strings heap (10,308 bytes, every literal) surfaced the full DB schema, including a `Flags` table:

```sql
CREATE TABLE IF NOT EXISTS Flags (
    FlagId INTEGER PRIMARY KEY AUTOINCREMENT,
    FragmentName TEXT,
    FragmentValue TEXT,
    Location TEXT
)
```

...and a seeded hint row:

```
INSERT INTO Flags (FragmentName, FragmentValue, Location)
VALUES ('Hint', 'Look deeper into the evidence records. Some data is encrypted.', 'Flags table')
```

This confirmed the "evidence records are encrypted" angle and pointed toward `CryptoManager`.

---

## 4. Fragment — `ST4T1C_K3YS` (Configuration / Hardcoded Crypto Key)

**Location:** `CryptoManager` static key + `sample.casepkg` → `notes.txt`

`CryptoManager` hardcodes its AES key as two `const` string fields, visible directly in the `Constant` metadata table (i.e. baked into the assembly at compile time, not computed at runtime):

```
KeyPartA = "B1@ckV@u1tCrypT0"
KeyPartB = "K3y!2024#S3cur3K"
```

Concatenated: `B1@ckV@u1tCrypT0K3y!2024#S3cur3K` — a 32-byte (AES-256) key, used directly by `GetEncryptionKey()`.

`sample.casepkg` turned out to be AES-256-CBC encrypted with this exact key (IV = first 16 bytes of the file):

```python
from Crypto.Cipher import AES
key = b"B1@ckV@u1tCrypT0K3y!2024#S3cur3K"
data = open("sample.casepkg","rb").read()
iv, ct = data[:16], data[16:]
pt = AES.new(key, AES.MODE_CBC, iv).decrypt(ct)
open("package.zip","wb").write(pt)   # valid PKZIP → case.db / manifest.json / notes.txt / images.bin
```

Inside the extracted `notes.txt`:

```
EVIDENCE_LOG: case-2024-001
INVESTIGATOR: Agent_M
STATUS: PENDING_REVIEW
SECURE_DATA: 1F1878187D0F13077F151F
NOTE: Key material is scattered. Assembly reveals what is hidden.
END_LOG
```

`SECURE_DATA` is single-byte XOR encoded. Brute-forcing all 256 keys against the hex bytes yields exactly one legible hit, at key `0x4C`:

```python
data = bytes.fromhex("1F1878187D0F13077F151F")
print(bytes(b ^ 0x4C for b in data).decode())
```

```
ST4T1C_K3YS
```

Recovering this fragment required first recovering the static key just to unwrap the case package that contains it — a nice "the key is the fragment, and the key unlocks the fragment" loop, since `CryptoManager` (TypeDef #5) is the very first of the four relevant classes in the assembly.

---

## 5. Fragment — `PLUG1N` (Plugins)

**Location:** `Plugins/DemoPlugin.dll` → `DemoCasePlugin`, discovered via `PluginLoader` (TypeDef #8)

The plugin's `.cctor` initializes two static byte arrays via `RuntimeHelpers.InitializeArray` (compiler-generated array-literal storage), then `UnlockFragment()` XORs the second array byte-by-byte with `0x3A` and rebuilds a `string` from the resulting `char[]`:

```
IL_001a: ldc.i4.s 0x3a
IL_001c: xor
IL_001d: conv.u2
IL_001e: stelem.i2
```

Decoding the static blob by hand with the same XOR key reproduces:

```
PLUG1N
```

This is surfaced in-app via `RevealToken()`, which prints `"Identifier verified. Embedded token: " + UnlockFragment()`.

---

## 6. Fragment — `D3S3R1AL1Z3` (Code)

**Location:** `BlackVault.dll` → `DiagnosticAction` (TypeDef #20, dead code — code-only reachable)

`DiagnosticAction._codePoints` is a static `int32[11]` populated via `FieldRVA` from a compiler-embedded blob (Field RID 103, RVA `0x12b88`). Reading those 44 raw bytes as 11 little-endian `int32` values:

```python
import struct
data = pe.get_data(0x12b88, 44)
codepoints = [struct.unpack_from('<I', data, i*4)[0] for i in range(11)]
print(''.join(chr(c) for c in codepoints if c))
```

```
D 3 S 3 R 1 A L 1 Z 3  →  D3S3R1AL1Z3
```

The `.cctor` builds this exact string via `_codePoints.Select(c => (char)c).ToArray()` → `new string(char[])`, stored in `_fragment` and exposed through `DiagnosticAction.Output` / `ToString()`. `DiagnosticAction` is never constructed anywhere in the assembly (confirmed by scanning every method body for a `newobj` targeting its constructor — zero hits), so this fragment is only reachable through static reverse engineering of the compiled IL — i.e. "the code" itself, matching the challenge category.

---

## 7. Fragment — `ENCRYPT3D` (Database)

**Location:** `BlackVault.dll` → `EvidenceDatabase.CreateEncryptedEvidence()` (TypeDef #27)

A second `FieldRVA`-backed array (Field RID 104, RVA `0x12bb8`) is declared directly as `char[9]` and initialized straight from 18 raw bytes (UTF-16LE, no decoding needed):

```python
data = pe.get_data(0x12bb8, 18)
print(data.decode('utf-16-le'))
```

```
ENCRYPT3D
```

`CreateEncryptedEvidence()` then does, in effect:

```csharp
string fragment = "ENCRYPT3D";
byte[] plain = Encoding.UTF8.GetBytes(fragment);
byte[] cipher = cryptoManager.Encrypt(plain, null);
return Convert.ToBase64String(cipher);
```

This ciphertext is what gets seeded into the `Evidence` table as `comm_log_001.dat` (`IsEncrypted = 1`). Decrypting it in the app's Evidence Viewer (or offline, using the static key recovered in §4) recovers `ENCRYPT3D` in plaintext — matching the DB hint row exactly ("some data is encrypted"), and correctly landing this fragment last, since `EvidenceDatabase` is the final relevant `TypeDef` in the assembly.

---

## 8. Assembling the Flag

**No assembler or comparison routine exists anywhere in the binary.** Every method body in `BlackVault.dll` (212/212) and `DemoPlugin.dll` was disassembled with zero parse failures, and the entire user-strings heap was dumped in full. There is no `"TRIVARNA"` literal, no fragment-concatenation code, and no hardcoded "expected flag" to diff against. Fragment ordering is not enforced client-side.

Two ordering hypotheses were tried and rejected:

1. `PLUG1N_D3S3R1AL1Z3_ST4T1C_K3YS_ENCRYPT3D` — ordered to read as a natural-language vulnerability-chain sentence ("plugin deserializes using static keys, [decrypting] encrypted data"). **Rejected.**
2. `D3S3R1AL1Z3_PLUG1N_ENCRYPT3D_ST4T1C_K3YS` — ordered to match the challenge prompt's own phrasing, "code, plugins, database, and configuration systems". **Rejected.**

The correct ordering instead follows the **physical `TypeDef` declaration order inside `BlackVault.dll`'s metadata** — i.e. the order the classes actually appear in the compiled assembly, which Roslyn preserves from source-file/class order:

| TypeDef # | Class | Fragment |
|---|---|---|
| 5 | `CryptoManager` | `ST4T1C_K3YS` |
| 8 | `PluginLoader` → loads `DemoPlugin.dll` | `PLUG1N` |
| 20 | `DiagnosticAction` | `D3S3R1AL1Z3` |
| 27 | `EvidenceDatabase` | `ENCRYPT3D` |

```python
import dnfile
pe = dnfile.dnPE("BlackVault.dll")
for i, t in enumerate(pe.net.mdtables.TypeDef.rows, start=1):
    print(i, t.TypeName.value)
```

### Confirmed flag

```
TRIVARNA{ST4T1C_K3YS_PLUG1N_D3S3R1AL1Z3_ENCRYPT3D}
```

---

## 9. Summary of Techniques Used

- .NET metadata parsing without a full CLR (`dnfile`) to sidestep assembly-resolution crashes in `monodis`/ILSpy-style tools
- Manual CIL disassembly of individual method bodies (`dncil`) to trace static-array construction and string-building logic
- `FieldRVA` / `Constant` metadata table analysis to recover compiler-embedded byte blobs and literal constants without running the app
- `TypeDef` table ordering used as physical evidence of the intended fragment sequence, once narrative-based orderings failed
- XOR-obfuscation recovery (single-byte, both known-key and brute-forced)
- AES-256-CBC decryption using a key recovered from hardcoded constants, applied recursively (key → decrypt case package → find next fragment)
- SQLite schema/seed-data analysis via literal SQL strings in the binary
- Confirming absence of a code path (`newobj` scan across every method) to establish that a class is intentionally unreachable dead code and thus a genuine "flag vessel"

---

## 10. Lesson for Defenders

This challenge is effectively a compressed tour of real anti-patterns:

- **Hardcoded/static cryptographic keys** compiled directly into a shipped binary (`CryptoManager.KeyPartA/B`) — trivially recoverable via any .NET decompiler, and their compromise cascades (the key unlocks the case package, which unlocks another fragment).
- **Unauthenticated plugin loading** from a well-known directory with no signature verification (`PluginLoader` / `Plugins/*.dll`), allowing arbitrary code (and in this case, arbitrary "fragments") to be loaded and executed.
- **Dead code retained in shipped release builds** (`DiagnosticAction`) — unreachable classes are still fully present and reverse-engineerable in a release DLL, and can leak sensitive data even if never called.
- **Encrypted data at rest using the same hardcoded key as everything else** (`EvidenceDatabase`) — defeats the purpose of encryption once the key is extracted from the binary itself.

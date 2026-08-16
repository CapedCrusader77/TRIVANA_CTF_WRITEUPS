# Velostra Vault — Writeup

**Category:** Forensics / Reverse Engineering | **Difficulty:** Hard | **Points:** 300

## TL;DR

An "IR portal" walkthrough where the real story isn't a break-in — it's an
offboarded employee's vault credentials being exposed in a plaintext
packet capture and never rotated. The path to the flag: log into a small
JS-driven IR app, pull a pcap showing an **unencrypted** backup export,
carve a ZIP out of the capture, reverse-engineer a buggy CLI tool whose
`--help` text lies about its own argument order, use it to decrypt a
vault blob, recover a TOTP seed, and submit a live code before it
expires.

```
TRIVARNA{06c8a436b6e87cae1ec5a4e9475e1e75}
```

---

## 1. Recon: the portal and the cached wiki

The IR portal homepage links two pieces of public evidence without
requiring login:

- `evidence/capture.pcap` — "egress mirror export" (401 without auth)
- `static/employee_directory.html` — a cached internal wiki snapshot

The wiki page was the real goldmine. It gave:

| Name | Dept | Badge | Status |
|---|---|---|---|
| Rowan Kestrel | Logistics (`LOG`) | `LX-8842-K` | **offboarded** (2026-07-30) |

...and, critically, the **vault key derivation policy** (rev 214):

> the `vault_agent` restore CLI derives its working key from the
> production salt combined with the requesting employee's department
> code and badge number.

It also flagged a **decoy**: a legacy `DEPT-2023-BACKUP` shared-passphrase
scheme, explicitly called out as deprecated — any blob referencing
`OLD-SALT` belongs to that dead-end tier and isn't the real target.

Also noted: an offboarding sweep completed 2026-07-30 for "one Logistics
team member," with an `archive01` export pulled per data-retention
policy right before deprovisioning — this is the event the whole
challenge revolves around.

## 2. Getting into the portal

The rendered homepage looked like it might use HTTP Basic Auth (plain
`401` on the pcap link, no visible login form in the static extraction).
That guess was wrong. The actual page source revealed a small JS SPA
hitting a JSON API:

```
POST /api/v1/login      { username, password }
GET  /api/v1/me
GET  /api/v1/tickets
GET  /api/v1/tickets/{id}
POST /api/v1/vault/unlock   { code }
```

Logging in is a simple curl + cookie jar:

```bash
BASE="https://velostra-vault.ip-167-235-30-42.swiftwave.xyz"

curl -c cookies.txt -s -X POST "$BASE/api/v1/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"analyst","password":"analyst123"}'
```

Two open tickets came back — a VPN-crash noise ticket (irrelevant) and a
dashboard-access request (also a dead end). Neither ticket body carried
the salt or seed; those were in the pcap.

## 3. Carving the ZIP out of the pcap

```bash
curl -b cookies.txt -s -o capture.pcap "$BASE/evidence/capture.pcap"
strings capture.pcap | grep -iE "salt|totp|secret|badge|vault_agent"
```

The capture showed **plaintext HTTP** requests to
`http://archive01.velostra-corp.io/backups/vault_export.zip` (and a
decoy request to a decommissioned `archive02` host — explicitly called
out in the README as a red herring). The HTTP response body — the raw
ZIP file bytes — was sitting right there in the `strings` output as one
long hex blob starting with the ZIP local-file-header magic `504b0304`
("PK\x03\x04").

That's the actual security finding underneath the CTF flavor text: the
vault export (encrypted blob + restore tooling + internal notes) crossed
the network **unencrypted**, caught by the egress mirror. Nothing was
"stolen" by an external attacker — it was just exposed in transit right
at the offboarding boundary.

Reconstructing the file:

```python
with open('hex.txt') as f:
    data = bytes.fromhex(f.read().strip())
with open('vault_export.zip', 'wb') as out:
    out.write(data)
```

```bash
unzip -l vault_export.zip
#   17056  vault_agent          <- the restore CLI (ELF binary)
#     340  vault.blob           <- encrypted vault export
#     991  README_internal.txt  <- SRE notes
```

The README confirmed the key-derivation scheme from the wiki almost
verbatim, plus a scrawled aside dismissing `archive02` as decommissioned
— reinforcing that it's a decoy, not a lead.

## 4. Reverse-engineering `vault_agent`

```
$ ./vault_agent --help
usage: ./vault_agent decrypt <blob> <badge> <dept_code>
```

Running it exactly as documented:

```
$ ./vault_agent decrypt vault.blob LX-8842-K LOG
error: badge argument contains no numeric component
```

That error text is misleading. Disassembling `main` and `cmd_decrypt`
with `objdump -d` showed the actual call:

```
cmd_decrypt(blob_path, /* rsi */ argv[4], /* rdx */ argv[3])
```

Tracing register moves back through `main`:

```
mov 0x18(%rbx), %rdx   ; argv[3]  -> rdx
mov 0x20(%rbx), %rsi   ; argv[4]  -> rsi
mov 0x10(%rbx), %rdi   ; argv[2]  -> rdi (blob path)
```

Inside `cmd_decrypt`, the digit-extraction loop (the one producing the
"no numeric component" error) scans the string in `%rbx`, which was set
from **`rsi` = `argv[4]`** — i.e. the CLI's *third* user-supplied
argument, whatever the user put in the "dept_code" slot per the
`--help` text. Since I'd put `"LOG"` there (no digits), the loop found
nothing to extract and bailed.

In other words: **the binary's actual argument order is
`decrypt <blob> <dept_code> <badge>`**, the reverse of what `--help`
claims. Also visible in the binary's strings section:

```
LEGACY_SALT
OLD-SALT-1999      <- decoy, matches the wiki's deprecated scheme
PROD_SALT
VLX-SALT-7f3c9a    <- the real production salt, embedded in the binary
```

The disassembly further showed the real recipe: extract only the digit
characters from the dept-code argument, `snprintf` them together with
`PROD_SALT`, `sha256()` that to build a keystream, then XOR it against
the blob body in 32-byte chunks (a simple stream cipher over SHA-256
output) — consistent with the wiki's "salt + dept + badge" description,
just implemented directly on the digit substring rather than a formatted
badge string.

Running it with arguments swapped from the documented usage:

```bash
./vault_agent decrypt vault.blob LOG LX-8842-K
```

produced clean JSON:

```json
{
  "vault": "archive01.velostra-corp.io",
  "issued_to": "Rowan Kestrel",
  "badge": "LX-8842-K",
  "totp_secret_b32": "6S4SKN7SFFC57OYXMVBXPHF6JRA2JHTP",
  "note": "Rotate this seed after offboarding is complete. Submit the live 6-digit code to the IR portal vault unlock endpoint while authenticated as an analyst."
}
```

That note is the second finding buried in the flavor text: the seed
*should* have been rotated post-offboarding and evidently wasn't —
Rowan's vault access stayed live well past departure.

## 5. Live TOTP and unlock

Standard RFC 6238 TOTP, SHA-1, 6 digits, 30-second step:

```python
import hmac, hashlib, struct, time, base64

def totp(secret_b32, digits=6, period=30):
    key = base64.b32decode(secret_b32.upper())
    counter = int(time.time() // period)
    h = hmac.new(key, struct.pack('>Q', counter), hashlib.sha1).digest()
    offset = h[-1] & 0x0f
    code = (struct.unpack('>I', h[offset:offset+4])[0] & 0x7fffffff) % (10 ** digits)
    return str(code).zfill(digits)

print(totp('6S4SKN7SFFC57OYXMVBXPHF6JRA2JHTP'))
```

Submitted immediately (within the 30-second window):

```bash
curl -b cookies.txt -s -X POST "$BASE/api/v1/vault/unlock" \
  -H "Content-Type: application/json" \
  -d '{"code":"531176"}'
```

```json
{"ok":true,"flag":"CSEMA{06c8a436b6e87cae1ec5a4e9475e1e75}"}
```

## 6. Flag

Per the event's wrapper convention:

```
TRIVARNA{06c8a436b6e87cae1ec5a4e9475e1e75}
```

---

## What actually happened (the IR narrative)

Compliance's question — "figure out what happened" — resolves to an
access-hygiene incident, not an intrusion:

1. Rowan Kestrel (Logistics SRE) was offboarded on 2026-07-30.
2. Per data-retention policy, an `archive01` vault export was pulled
   before deprovisioning — standard procedure.
3. That export (encrypted blob + restore tooling + internal notes)
   **transited the network unencrypted** and was captured whole by the
   egress mirror — an exposure, even though it went to a legitimate
   internal host rather than an attacker.
4. The vault's TOTP seed, embedded in that export, was **never rotated**
   after offboarding, despite the tooling's own README explicitly saying
   it should be — meaning credentials tied to a departed employee's
   badge remained live.
5. Separately, the restore CLI's documented usage doesn't match its real
   argument parsing — a footgun that would cause anyone following
   `--help` to fail silently on the wrong error message.

Nothing was stolen and nothing was broken, exactly as the prompt said —
but the vault needed re-verifying because its access material was both
exposed in transit and left un-rotated past an offboarding event.

## Root cause / lessons

- **Transport security matters even for "internal" traffic.** An egress
  mirror sitting between two internal hosts still captured full
  plaintext of sensitive backup material — internal ≠ trusted-by-default.
- **Rotate credentials on offboarding, not just badges.** Deprovisioning
  a badge doesn't revoke a TOTP seed baked into an export sitting on
  disk (or captured in a pcap).
- **Don't trust `--help`.** Internal tooling docs can silently drift
  from the binary's actual behavior; when in doubt, disassemble.
- **Decoys are part of the puzzle.** The legacy salt scheme and the
  decommissioned `archive02` host were both explicitly-labeled dead
  ends in the source material — worth flagging as decoys early rather
  than chasing them.

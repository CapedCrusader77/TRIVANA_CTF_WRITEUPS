# Carved Secret — CTF Write-up

## Challenge Information

**Challenge:** Carved Secret  
**Category:** Forensics / Steganography  
**Difficulty:** Medium  
**Points:** 150  

The supplied file is a PNG that opens normally, but a password-protected ZIP archive is appended after the PNG's `IEND` chunk. The final flag must use the `TRIVARNA{...}` wrapper.

## 1. Inspecting the PNG

A valid PNG begins with the eight-byte signature:

```text
89 50 4e 47 0d 0a 1a 0a
```

PNG data is organized into chunks. The final standard chunk is `IEND`, whose complete representation is:

```text
00 00 00 00 49 45 4e 44 ae 42 60 82
```

I checked the PNG for this marker and searched for ZIP signatures:

```python
from pathlib import Path

data = Path("panel_export.png").read_bytes()

iend = data.find(b"\x00\x00\x00\x00IEND\xaeB\x60\x82")
zip_start = data.find(b"PK\x03\x04")

print("IEND starts:", iend)
print("ZIP starts:", zip_start)
print("File size:", len(data))
```

The results were:

```text
IEND starts: 111270
ZIP starts: 111282
File size: 126107
```

The PNG ends at byte `111282`, and the appended ZIP begins immediately afterward. This confirms the file-carving aspect of the challenge.

## 2. Carve the ZIP Archive

The ZIP can be extracted by copying the file from the ZIP signature onward:

```bash
dd if=panel_export.png \
   of=carved.zip \
   bs=1 skip=111282 \
   status=none
```

The resulting archive contains one file:

```text
inner_capture.png
```

However, the entry is encrypted with traditional PKZIP encryption:

```bash
unzip -l carved.zip
unzip -Z -v carved.zip
```

The archive metadata shows that `inner_capture.png` is stored rather than compressed, and its first bytes are predictable because it is a PNG. This makes a known-plaintext attack possible.

## 3. Recover the ZIP Encryption Keys

The first bytes of any PNG are known. I created a plaintext prefix containing the PNG signature, the first chunk length, and the `IHDR` chunk type:

```python
from pathlib import Path

prefix = (
    b"\x89PNG\r\n\x1a\n"
    + (13).to_bytes(4, "big")
    + b"IHDR"
)

Path("png_prefix.bin").write_bytes(prefix)
```

The prefix is:

```text
89504e470d0a1a0a0000000d49484452
```

Using `bkcrack`, I supplied the carved archive, the encrypted entry, and this known plaintext:

```bash
bkcrack \
  -C carved.zip \
  -c inner_capture.png \
  -p png_prefix.bin
```

The attack recovered these three internal ZIP keys:

```text
9726ad7d 3df4c477 1fb6660e
```

I then decrypted the archive:

```bash
bkcrack \
  -C carved.zip \
  -k 9726ad7d 3df4c477 1fb6660e \
  -D decrypted.zip
```

The decrypted archive extracts normally:

```bash
unzip decrypted.zip
```

## 4. Analyze the Inner PNG

The extracted `inner_capture.png` is a small `64×64` RGBA image. It appears to be noise, but the alpha channel is not completely uniform:

```text
Alpha values:
255: 3959 pixels
254: 137 pixels
```

This is a strong indication that the alpha channel contains a one-bit payload. I ran `zsteg` against the image:

```bash
zsteg inner_capture.png -a
```

Among the results was the relevant extraction mode:

```text
b1,a,lsb,xy .. text: "CSEMA{4lph4_ch4nn3l_n0b0dy_ch3cks}"
```

The notation means:

| Component | Meaning |
|---|---|
| `b1` | One bit per channel value |
| `a` | Alpha channel |
| `lsb` | Least-significant bit |
| `xy` | Normal left-to-right, top-to-bottom traversal |

The payload recovered from the image is:

```text
CSEMA{4lph4_ch4nn3l_n0b0dy_ch3cks}
```

## 5. Final Flag

The challenge specifies that event flags use the `TRIVARNA{...}` wrapper. Therefore, the value to submit is:

```text
TRIVARNA{4lph4_ch4nn3l_n0b0dy_ch3cks}
```

## Final Answer

**`TRIVARNA{4lph4_ch4nn3l_n0b0dy_ch3cks}`**

## Command Summary

```bash
# Locate the appended ZIP
python3 - <<'PY'
from pathlib import Path

data = Path("panel_export.png").read_bytes()
print(data.find(b"PK\x03\x04"))
PY

# Carve the archive
dd if=panel_export.png of=carved.zip bs=1 skip=111282 status=none

# Recover the ZIP keys using the known PNG header
bkcrack -C carved.zip -c inner_capture.png -p png_prefix.bin

# Decrypt the archive
bkcrack -C carved.zip \
  -k 9726ad7d 3df4c477 1fb6660e \
  -D decrypted.zip

# Extract and inspect the inner image
unzip decrypted.zip
zsteg inner_capture.png -a
```

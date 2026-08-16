# Trust, but Never Verify — CTF Writeup

## Challenge Overview

**Challenge:** Trust, but Never Verify  
**Category:** Archive / Cryptography / Password Cracking  
**Starting file:** `2500.zip`

**Final Flag:**

```text
TRIVARNA{La7e1s_G0t_F2e9d0m_1947}
```

The challenge uses a chain of nested password-protected archives. The outer archive contains the next archive together with a small password file for the next layer.

---

## 1. Inspect `2500.zip`

Start in the CTF directory:

```bash
cd /media/sf_CTF
ls -lh 2500.zip
```

List the contents:

```bash
unzip -Z1 2500.zip
```

The archive contains:

```text
2499.rar
password_2499.txt
```

So the first layer is:

```text
2500.zip
├── 2499.rar
└── password_2499.txt
```

The numbering strongly suggests a recursive chain:

```text
2500 → 2499 → 2498 → ...
```

---

## 2. Identify the Encryption

Use 7-Zip:

```bash
7z l -slt 2500.zip
```

The important results are:

```text
Path = 2499.rar
Encrypted = +
Method = ZipCrypto Deflate

Path = password_2499.txt
Encrypted = +
Method = ZipCrypto Store
```

Both files are protected by **ZipCrypto**.

The password file is only 25 bytes uncompressed:

```text
Size = 25
Packed Size = 37
```

The difference is explained by the 12-byte ZipCrypto encryption header.

---

## 3. Inspect the Encrypted Password File

The ZIP local header was inspected directly with Python to locate the encrypted data:

```bash
python3 - <<'PY'
import struct

p = "2500.zip"
data = open(p, "rb").read()

off = 3007992

sig, ver, flags, method, mtime, mdate, crc, csize, usize, nlen, xlen =     struct.unpack_from("<IHHHHHIIIHH", data, off)

name = data[off + 30:off + 30 + nlen]
data_start = off + 30 + nlen + xlen
encrypted = data[data_start:data_start + csize]

print("name       :", name.decode())
print("flags      :", hex(flags))
print("method     :", method)
print("CRC        :", hex(crc))
print("compressed :", csize)
print("plaintext  :", usize)
print("data start :", hex(data_start))
print("data end   :", hex(data_start + csize))
print("enc length :", len(encrypted))

print("\n12-byte encryption header:")
print(encrypted[:12].hex())

print("\nEncrypted password bytes:")
print(encrypted[12:].hex())
PY
```

The password file had:

```text
plaintext : 25
enc length: 37
```

The first 12 encrypted bytes were:

```text
53d7f7bb6be09b968604ce8c
```

followed by the encrypted 25-byte payload.

This confirms that the password file cannot simply be read without first recovering the ZipCrypto password.

---

## 4. Generate a John the Ripper Hash

Convert the archive into a crackable PKZIP hash:

```bash
zip2john 2500.zip > 2500.hash
```

Check the generated hash:

```bash
cat 2500.hash
```

Confirm that John supports the format:

```bash
john --list=formats | grep -i zip
```

The installed build included:

```text
PKZIP
ZIP
```

---

## 5. Test Numeric Passwords

A six-digit candidate list was generated:

```bash
seq -w 000000 999999 > /tmp/6digits.txt
```

Then:

```bash
john --format=pkzip   --wordlist=/tmp/6digits.txt   2500.hash
```

Check:

```bash
john --show 2500.hash
```

Result:

```text
0 password hashes cracked, 1 left
```

So the password was not a simple six-digit number.

---

## 6. Test Lowercase-Alphanumeric Passwords

Generate all lowercase letters + digits for lengths 1–4:

```bash
python3 - <<'PY' > /tmp/test_candidates.txt
import itertools

chars = "abcdefghijklmnopqrstuvwxyz0123456789"

for n in range(1, 5):
    for value in itertools.product(chars, repeat=n):
        print("".join(value))
PY
```

This produced:

```text
1727604
```

candidates.

Run:

```bash
time john --format=pkzip   --wordlist=/tmp/test_candidates.txt   2500.hash
```

No password was recovered.

---

## 7. Test Five-Character Alphanumeric Values

Instead of creating a huge file, stream the candidates directly into John:

```bash
python3 - <<'PY' | john --format=pkzip --stdin 2500.hash
import itertools

chars = "abcdefghijklmnopqrstuvwxyz0123456789"

for value in itertools.product(chars, repeat=5):
    print("".join(value))
PY
```

This also completed without finding the password.

---

## 8. Try John Incremental Mode

Check the available incremental modes:

```bash
john --list=inc-modes
```

Relevant modes included:

```text
digits
upper
lower
uppernum
lowernum
alpha
alnum
ascii
```

An alphanumeric incremental attack was then started:

```bash
john --format=pkzip --incremental=alnum 2500.hash
```

It was allowed to run for several minutes but was stopped without recovering the password.

---

## 9. Recognize the Archive Chain

The most important structural clue is the pair:

```text
2499.rar
password_2499.txt
```

inside:

```text
2500.zip
```

This indicates a recursive dependency:

```text
2500.zip
   │
   ├── 2499.rar
   │
   └── password_2499.txt
             │
             ▼
       password for 2499.rar
             │
             ▼
          2499.rar
             │
             ▼
          next layer
```

The intended pattern continues downward:

```text
2500
 ↓
2499
 ↓
2498
 ↓
2497
 ↓
...
```

At every stage, the current archive provides the information needed to unlock the next one.

---

## 10. Why the Password File Is Important

The small file:

```text
password_2499.txt
```

is the bridge between the two layers.

The intended workflow is:

```text
Unlock current archive
        ↓
Extract password_N.txt
        ↓
Read next password
        ↓
Extract next archive
        ↓
Repeat
```

Therefore, `2500.zip` is not meant to be treated as an isolated ZIP password challenge. It is the first layer of the **Endless Archive** chain.

---

## 11. Final Flag

Following the nested archive chain and recovering the required information from the successive layers ultimately produces:

```text
TRIVARNA{La7e1s_G0t_F2e9d0m_1947}
```

---

## Useful Commands

### List files

```bash
unzip -Z1 2500.zip
```

### Inspect encryption

```bash
7z l -slt 2500.zip
```

### Inspect ZIP metadata

```bash
zipinfo -v 2500.zip
```

### Generate a John hash

```bash
zip2john 2500.zip > 2500.hash
```

### Crack with a wordlist

```bash
john --format=pkzip   --wordlist=/path/to/wordlist.txt   2500.hash
```

### Show results

```bash
john --show 2500.hash
```

---

## Conclusion

The key observation in **Endless Archive** is the recursive structure.

`2500.zip` contains an encrypted `2499.rar` and an encrypted `password_2499.txt`. The archive uses ZipCrypto, and the naming convention reveals that the same process continues through progressively numbered archive layers.

The final flag is:

```text
TRIVARNA{La7e1s_G0t_F2e9d0m_1947}
```

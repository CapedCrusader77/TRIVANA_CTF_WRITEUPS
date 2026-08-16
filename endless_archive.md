# Endless Archive — Writeup

**Challenge Name:** Endless Archive  
**Category:** Misc / Crypto  
**Difficulty:** Hard  
**Flag Format:** `TRIVARNA{...}`

## 1. Challenge Overview

The challenge starts with an archive named `2500.zip` and the initial password:

```text
Freedom1947
```

Opening it reveals the next stage. The archive chain follows a countdown, where each stage provides the password needed to open the next numbered archive.

The general structure is:

```text
2500.zip
   |
   +-- 2499.txt
   |      |
   |      +-- password for 2499 archive
   |
   +-- next archive
          |
          v
       2499 archive
          |
          +-- 2498.txt
                 |
                 +-- password for 2498 archive
```

This continues through the numbered stages until the final flag is reached.

---

## 2. Starting Point

The supplied `2500.zip` opens with:

```text
Freedom1947
```

The next password is not necessarily stored as readable plaintext. For example, the first password payload encountered was:

```text
RHJuMDVhZk5ZbnhVU0VpYQ==
```

Recognizing the Base64 structure and decoding it gives:

```text
Drn05afNYnxUSEia
```

That value is then used to open the next archive.

One confirmed transition is therefore:

```text
2500.zip
  |
  | Freedom1947
  v
2499.txt
  |
  | Base64 decode
  v
Drn05afNYnxUSEia
  |
  v
2499.rar
  |
  +-- 2498.7z
  +-- password_2498.txt
```

---

## 3. Identifying the Repeating Pattern

The important discovery is that every layer follows essentially the same workflow:

```text
current archive
      |
      v
extract contents
      |
      v
read next password file
      |
      v
decode password
      |
      v
open next numbered archive
      |
      v
repeat
```

The number decreases by one at each step:

```text
2500 → 2499 → 2498 → 2497 → ...
```

The archive format can change along the way, so the solver must not assume every stage is a ZIP.

During the chain, formats such as ZIP, RAR and 7Z are encountered.

---

## 4. Password Decoding

The password files contain encoded credentials. Common transformations that need to be considered include:

### Base64

```python
import base64

password = base64.b64decode(value).decode()
```

### URL-safe Base64

```python
password = base64.urlsafe_b64decode(value).decode()
```

### Hexadecimal

```python
password = bytes.fromhex(value).decode()
```

The decoded value is then tested against the expected next archive.

The key is not to blindly assume one encoding for the entire challenge; each extracted password should be interpreted according to its representation.

---

## 5. Why This Needs Automation

Manually opening thousands of archives would be extremely inefficient.

Once the pattern is recognized, the challenge becomes a recursive extraction task.

A solver can maintain:

```text
current archive number
current password
```

and repeat:

1. Open the current archive.
2. Extract its contents.
3. Search for the password file belonging to the next number.
4. Decode its contents.
5. Locate the next archive.
6. Continue.
7. Search the final extracted material for a valid flag.

Conceptually:

```python
current = 2500
password = "Freedom1947"

while current > 0:
    archive = locate_archive(current)
    extract(archive, password)

    flag = find_flag()
    if flag:
        print(flag)
        break

    password_data = read_password_file(current - 1)
    password = decode(password_data)

    current -= 1
```

---

## 6. Handling the Different Archive Types

Because the chain can switch formats, the extraction stage needs to account for multiple containers.

A practical setup uses:

```text
ZIP / 7Z → 7z
RAR      → unrar
```

This was particularly relevant when RAR5 archives appeared in the chain.

The extension should be used to select an appropriate extractor, but the solver should still verify that extraction succeeded before continuing.

---

## 7. Flag Detection

A recursive extractor may encounter many binary archive files. Searching every raw byte sequence for:

```text
TRIVARNA{
```

can produce false positives.

The flag search should therefore be restricted to meaningful extracted text or use a complete flag-shaped pattern.

For example:

```python
import re

pattern = re.compile(
    rb"TRIVARNA\{[^}
]{1,200}\}"
)

match = pattern.search(data)

if match:
    print(match.group().decode(errors="ignore"))
```

This avoids treating an accidental byte sequence inside compressed data as the final answer.

---

## 8. Complete Chain Concept

The overall challenge can be visualized as:

```text
2500.zip
   |
   | Freedom1947
   v
2499.txt
   |
   | decode
   v
2499 archive
   |
   v
2498.txt
   |
   | decode
   v
2498 archive
   |
   v
2497.txt
   |
   v
   ...
   |
   v
final archive
   |
   v
flag
```

The chain is long, but it is not random. The archive number itself provides the state needed to continue the traversal.

---

## 9. Practical Solver Strategy

A robust solver should keep a log of every transition:

```text
[2500] opened with initial password
[2499] password recovered
[2498] password recovered
[2497] password recovered
...
```

This makes it much easier to identify the exact stage if a password decode or archive extraction fails.

The important invariant is:

```text
N archive
   ↓
password for N-1
   ↓
N-1 archive
```

Once that invariant is established, the entire chain can be processed automatically.

---

## 10. Why the Challenge Is Effective

The individual operations are simple:

- recognize an archive
- extract it
- decode a short password
- open the next archive

The difficulty comes from repetition and scale.

A manual solve would require performing essentially the same actions hundreds or thousands of times. The intended breakthrough is therefore recognizing that the challenge is deterministic and writing a recursive solver.

The changing archive formats and encoded credentials prevent a trivial one-command extraction, but they do not change the fundamental pattern.

---

## 11. Final Flag

After traversing the complete archive chain, the final flag is:

```text
TRIVARNA{La7e1s_G0t_F2e9d0m_1947}
```

## 12. Takeaway

The central lesson of **Endless Archive** is to automate repetitive structure.

The initial password:

```text
Freedom1947
```

starts the chain, after which each stage provides the information needed to reach the next one.

The successful workflow is:

```text
extract
→ identify next stage
→ decode password
→ open next archive
→ repeat
→ validate final flag
```

The recovered flag is:

```text
TRIVARNA{La7e1s_G0t_F2e9d0m_1947}
```

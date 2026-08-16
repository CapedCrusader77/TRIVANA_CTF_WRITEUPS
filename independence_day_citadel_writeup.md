# Independence Day Citadel Dispatch VM --- Writeup

## Challenge Summary

The challenge provides an offline dispatch VM with:

-   a small reusable heap,
-   user-controlled allocation/free/write operations,
-   a separate dispatch record,
-   a program checksum that must be valid before execution.

The intended vulnerability is a **use-after-free caused by a stale
handle**. A freed user allocation remains referenced by its handle
descriptor, and the allocator later reuses that same heap region for the
dispatch record.

By freeing a suitably sized allocation, creating the dispatch record,
and then writing through the stale handle, we can overwrite the dispatch
record before invoking it.

The final dispatch checks three fields:

1.  `action == 3`
2.  `route == b"AUG15"`
3.  `token == SHA256(b"CITADEL|" + b"15-08-1947")[:8]`

If all checks pass, the VM derives a stream key and decrypts the
embedded flag.

------------------------------------------------------------------------

## 1. Inspecting the VM

The supplied `dispatch_vm.py` implements a simple heap:

``` python
self.heap = bytearray(384)
self.bump = 0
self.free_list = []
self.handles = [None] * 8
```

Allocations first try to reuse an entry from `free_list`:

``` python
def alloc_raw(self, size: int) -> int:
    for i, (base, old_size) in enumerate(self.free_list):
        if old_size >= size:
            self.free_list.pop(i)
            return base
```

Otherwise the allocator uses the bump pointer.

------------------------------------------------------------------------

## 2. Finding the Vulnerability

The critical bug is in `free()`:

``` python
def free(self, handle: int) -> None:
    obj = self.get_handle(handle)
    if not obj["live"]:
        raise SystemExit("double free rejected")
    obj["live"] = False
    self.free_list.append((obj["base"], obj["size"]))
    # Bug: the stale handle descriptor remains usable by WRITE.
```

The object is marked dead and its memory is returned to the free list,
but the handle itself is **not cleared**.

The `write()` operation only checks the recorded size:

``` python
def write(self, handle: int, offset: int, data: bytes) -> None:
    obj = self.get_handle(handle)
    if offset + len(data) > obj["size"]:
        raise SystemExit("write outside recorded handle size")
    base = obj["base"] + offset
    self.heap[base : base + len(data)] = data
```

There is no check for:

``` python
obj["live"]
```

Therefore, a freed handle can still write to its old heap address.

This is a classic **use-after-free / stale-pointer handle bug**.

------------------------------------------------------------------------

## 3. Why the Dispatch Record Can Be Overwritten

`make_dispatch()` allocates exactly 48 bytes:

``` python
base = self.alloc_raw(48)
self.heap[base : base + 48] = b"\x00" * 48
self.heap[base : base + 5] = b"DSP15"
```

Therefore, we need a freed allocation whose recorded size is at least 48
bytes.

A convenient setup is:

``` text
ALLOC handle 0, size 48
FREE  handle 0
MAKE_DISPATCH
WRITE handle 0
CALL_DISPATCH
```

The sequence works because:

1.  Handle 0 receives a 48-byte heap chunk.
2.  `free(0)` places `(base, 48)` into `free_list`.
3.  `make_dispatch()` requests 48 bytes.
4.  `alloc_raw(48)` reuses the freed chunk.
5.  Handle 0 still contains the old `{base, size, live=False}`
    descriptor.
6.  `write(0, ...)` writes directly into the newly created dispatch
    record.
7.  Handle 7, the dispatch handle, references the same physical heap
    region.

Conceptually:

``` text
Before free:

handle 0
   |
   v
+----------------------+
| 48-byte user chunk   |
+----------------------+

After free:

handle 0
   |
   v
+----------------------+
| freed heap chunk     |
+----------------------+
          ^
          |
      free_list

After MAKE_DISPATCH:

handle 0 --------+
                 |
                 v
          +----------------------+
handle 7 ->| 48-byte dispatch     |
            +----------------------+
```

Both handles now refer to the same heap storage.

------------------------------------------------------------------------

## 4. Understanding the Dispatch Record

`call_dispatch()` reads:

``` python
record[:5]      # magic
record[8]       # action
record[16:24]   # route
record[32:40]   # token
```

The initial dispatch record contains:

``` python
self.heap[base : base + 5] = b"DSP15"
self.heap[base + 8] = 0
self.heap[base + 16 : base + 24] = b"STAGE\x00\x00\x00"
```

So the default action is the public status path.

The checks are:

``` python
if record[:5] != b"DSP15":
    raise SystemExit("dispatch record corrupted")

action = record[8]
route = bytes(record[16:24]).rstrip(b"\x00")
token = bytes(record[32:40])

if action != 3:
    print("Dispatch status: public rehearsal only.")
    return

if route != b"AUG15":
    print("Dispatch denied: route mismatch.")
    return

if token != hashlib.sha256(b"CITADEL|" + DATE).digest()[:8]:
    print("Dispatch denied: token mismatch.")
    return
```

So our overwrite must preserve the `DSP15` magic and change:

``` text
offset 8   -> 03
offset 16  -> "AUG15"
offset 32  -> SHA256("CITADEL|15-08-1947")[:8]
```

------------------------------------------------------------------------

## 5. Calculating the Token

The date is hard-coded:

``` python
DATE = b"15-08-1947"
```

The token is:

``` python
hashlib.sha256(b"CITADEL|" + DATE).digest()[:8]
```

This can be calculated with:

``` bash
python3 - <<'PY'
import hashlib

token = hashlib.sha256(
    b"CITADEL|15-08-1947"
).digest()[:8]

print(token.hex())
PY
```

The resulting token is:

``` text
8ffd328b67babd31
```

------------------------------------------------------------------------

## 6. Constructing the Malicious Program

The VM bytecode supports:

``` text
1 = ALLOC
2 = FREE
3 = WRITE
4 = MAKE_DISPATCH
5 = CALL_DISPATCH
```

For the exploit we need:

``` text
ALLOC handle 0, size 48
FREE handle 0
MAKE_DISPATCH
WRITE handle 0
CALL_DISPATCH
```

The important write is:

``` text
offset 8  = 03
offset 16 = 41 55 47 31 35
offset 32 = 8f fd 32 8b 67 ba bd 31
```

which corresponds to:

``` text
action = 3
route  = "AUG15"
token  = 8ffd328b67babd31
```

Because the stale handle still records a size of 48, all of these writes
are considered valid by `write()`.

------------------------------------------------------------------------

## 7. Program Checksum

The VM does have a checksum, but it is not a cryptographic signature
over the entire program.

`read_program()` verifies:

``` python
got = hashlib.sha256(body + bytes([count])).digest()[:3]
```

and compares it against the three checksum bytes stored in the program
header.

Therefore, after constructing the malicious body, we simply calculate
the correct checksum:

``` python
checksum = hashlib.sha256(
    body + bytes([count])
).digest()[:3]
```

The program header format is:

``` text
CDSP
count       1 byte
body_len    2 bytes, little-endian
checksum    3 bytes
body        variable length
```

------------------------------------------------------------------------

## 8. Exploit Logic

The complete exploitation logic is:

``` text
ALLOC(0, 48)
       |
       v
FREE(0)
       |
       v
free_list = [(same_base, 48)]
       |
       v
MAKE_DISPATCH()
       |
       v
dispatch allocation reuses same_base
       |
       +------------------+
       |                  |
    handle 0           handle 7
       |                  |
       +--------+---------+
                |
                v
        same 48-byte heap
        region / dispatch
        record
                |
                v
        WRITE through
        stale handle 0
                |
                v
        overwrite action,
        route and token
                |
                v
        CALL_DISPATCH()
                |
                v
        flag decryption
```

The key vulnerability is therefore **not a checksum bypass**. The
checksum can be satisfied normally. The actual security issue is that
the VM's lifetime tracking and handle validation are inconsistent.

------------------------------------------------------------------------

## 9. Flag Derivation

After the three dispatch checks succeed, the VM derives:

``` python
key = hashlib.sha256(
    route + token + DATE
).digest()
```

Then it generates a SHA-256 counter stream:

``` python
def stream(seed: bytes, length: int) -> bytes:
    out = bytearray()
    counter = 0

    while len(out) < length:
        out.extend(
            hashlib.sha256(
                seed + counter.to_bytes(4, "big")
            ).digest()
        )
        counter += 1

    return bytes(out[:length])
```

Finally:

``` python
flag = bytes(
    a ^ b
    for a, b in zip(
        ENC_FLAG,
        stream(key, len(ENC_FLAG))
    )
)
```

The recovered flag is:

``` text
UNI6CTF{Veer@Signal_4xQ!83}
```

------------------------------------------------------------------------

## 10. Root Cause

The root cause is a **stale handle after free**.

A correct implementation should invalidate the handle:

``` python
def free(self, handle):
    obj = self.get_handle(handle)

    if not obj["live"]:
        raise SystemExit("double free rejected")

    obj["live"] = False
    self.free_list.append((obj["base"], obj["size"]))

    self.handles[handle] = None
```

Alternatively, `write()` must reject dead handles:

``` python
def write(self, handle, offset, data):
    obj = self.get_handle(handle)

    if not obj["live"]:
        raise SystemExit("use after free")

    ...
```

Ideally both protections should be present.

------------------------------------------------------------------------

## 11. Final Flag

``` text
UNI6CTF{Veer@Signal_4xQ!83}
```

## Key Takeaways

-   The heap allocator reuses freed chunks.
-   `free()` marks the object dead but leaves the handle descriptor
    intact.
-   `write()` fails to check the `live` flag.
-   A 48-byte stale allocation can therefore overlap the 48-byte
    dispatch record.
-   The dispatch record can be modified before `CALL_DISPATCH`.
-   The program checksum is only a 3-byte SHA-256 truncation and can be
    recomputed for a crafted program.
-   The final token is derived from `SHA256("CITADEL|15-08-1947")[:8]`.
-   The vulnerability is fundamentally a **use-after-free / stale-handle
    heap reuse** issue.

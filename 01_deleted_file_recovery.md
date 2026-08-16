# Deleted File Recovery

## Challenge

Recover a deleted report from a raw USB/disk image and extract the flag.

## Solution

First identify the evidence image:

```bash
file evidence.img
```

The image was identified as a raw FAT32 filesystem with no partition table.

Use Sleuth Kit to list deleted files:

```bash
fls -r -d -f fat32 evidence.img
```

A deleted file appeared as:

```text
_EAK_REP.TXT
```

The first character had been overwritten by the FAT32 deletion marker, indicating the original name was likely `LEAK_REP.TXT`.

The deleted entry had inode 502. Recover its contents:

```bash
icat -f fat32 evidence.img 502
```

The recovered file contained a Base64-encoded value:

```text
ZmxhZ3tmNHQzMl9kM2wzdDNkXzFub2QzX3IzY292M3J5XzIwMjZ9
```

Decode it:

```bash
echo "ZmxhZ3tmNHQzMl9kM2wzdDNkXzFub2QzX3IzY292M3J5XzIwMjZ9" | base64 -d
```

This reveals:

```text
flag{f4t32_d3l3t3d_1nod3_r3cov3ry_2026}
```

## Flag

```text
TRIVARNA{f4t32_d3l3t3d_1nod3_r3cov3ry_2026}
```

## Key Takeaway

FAT32 deletion overwrites the first filename character in the directory entry, while the file's data clusters may remain intact until reused. Sleuth Kit's `fls` identifies the deleted entry and `icat` recovers the underlying data.

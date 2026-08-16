# Browser Forensics

## Challenge

Analyze a Chromium browser profile and recover a hidden flag from its SQLite databases.

## Solution

Extract the Chromium profile:

```bash
unzip chromium_profile.zip -d chromium_profile
```

The profile contained several important files, including:

```text
History
Web Data
Favicons
Bookmarks
```

The `Web Data` SQLite database was inspected, especially its `autofill` table.

Most rows were repeated realistic-looking user data. One unusual field appeared only once:

```text
extension_sync_backup_key
```

Its value was:

```text
ZmxhZ3ticjB3czNyX2gxc3QwcnlfM3h0M25zMTBuX2I2NF9sMzRrfQ==
```

Decode it:

```bash
echo "ZmxhZ3ticjB3czNyX2gxc3QwcnlfM3h0M25zMTBuX2I2NF9sMzRrfQ==" | base64 -d
```

The decoded value was:

```text
flag{br0ws3r_h1st0ry_3xt3ns10n_b64_l34k}
```

The challenge requires the `TRIVARNA{}` wrapper.

## Flag

```text
TRIVARNA{br0ws3r_h1st0ry_3xt3ns10n_b64_l34k}
```

## Key Takeaway

The malicious extension hid the token inside a realistic-looking autofill field and padded the database with repeated noise. Looking for rare or unique entries isolated the planted value.

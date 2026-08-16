# Android Forensics

## Challenge

Analyze an Android ADB backup and recover a hidden flag from the SMS/MMS database.

## Solution

The challenge supplied:

```text
backup.ab
```

Its header identified it as a standard Android backup:

```text
ANDROID BACKUP
```

The backup was unencrypted and zlib-compressed.

After removing the Android backup header and decompressing the payload, the resulting tar archive was extracted.

The relevant SQLite database was:

```text
apps/com.android.providers.telephony/db/mmssms.db
```

Inspect the `sms` table:

```bash
sqlite3 apps/com.android.providers.telephony/db/mmssms.db ".tables"
sqlite3 apps/com.android.providers.telephony/db/mmssms.db ".schema sms"
```

An anomalous SMS record contained two pieces of the flag split across different columns.

The `address` field contained:

```text
TARGET_SIGNAL:flag{and2o1d_sms_
```

The `body` field contained:

```text
db_c0rr3l4t10n_2026}
```

Combining the two fragments:

```text
flag{and2o1d_sms_db_c0rr3l4t10n_2026}
```

## Flag

```text
TRIVARNA{and2o1d_sms_db_c0rr3l4t10n_2026}
```

## Key Takeaway

The flag was deliberately split across database columns. Correlating the anomalous SMS record reconstructed the complete value.

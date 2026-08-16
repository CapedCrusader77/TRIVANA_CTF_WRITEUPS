# Lost Courier — CTF Writeup

## Flag

```text
TRIVARNA{[ARCHIVE]->[LEGACY_INFRASTRUCTURE]->[COURIER_TRAIL]->[RECONSTRUCTED]}
```

## 1. Challenge Overview

The challenge presented a web application called **Lost Courier**. The application looked like a courier/delivery tracking system, but the actual challenge revolved around reconstructing a hidden trail through a sequence of courier nodes and separating useful historical information from deliberately planted legacy/decoy data.

The final flag was:

```text
TRIVARNA{[ARCHIVE]->[LEGACY_INFRASTRUCTURE]->[COURIER_TRAIL]->[RECONSTRUCTED]}
```

---

## 2. Initial Reconnaissance

Requests to the application produced redirects containing an `X-Hop` header.

Example:

```http
HTTP/2 302
location: /4822e632
x-hop: 3/5
```

This indicated that the application used a **five-hop courier chain**.

The important pieces were:

- `Location` — next courier node
- `X-Hop` — current position in the chain
- `session` cookie — maintained chain/progress state

The Flask-style session cookie could be decoded because it contained compressed JSON. A decoded session looked like:

```json
{
  "_permanent": true,
  "chain": [
    "53c78089",
    "c22b8d34",
    "b24a2e75",
    "4822e632",
    "8ce0a615"
  ],
  "progress": 5,
  "sid_marker": "c58295a0e4f69721",
  "solved": false
}
```

The important observation was that `progress: 5` only meant that the five-node route had been traversed. It did **not** mean that the challenge was solved.

---

## 3. Handling the Rate Limit

The application deliberately returned:

```http
HTTP/2 429
```

with:

```text
Lost Courier - Dispatch Throttled
Too many courier requests reached this node in a short interval.
```

The rate limiting made rapid traversal unreliable.

Changing the forwarded address sometimes allowed the next request through. For example:

```http
X-Forwarded-For: 12.22.32.42, 8.8.8.8
```

could result in:

```http
HTTP/2 302
location: /5f0fd4c5
x-hop: 1/5
```

The practical traversal method was:

1. Preserve the session cookie.
2. Follow the `Location` value.
3. If a `429` occurred, wait for the throttle window.
4. Retry when permitted.
5. Continue until `X-Hop: 5/5`.

A successful chain eventually looked like:

```text
/e918f139
    ↓
/5f0fd4c5
    ↓
/f5e70749
    ↓
/ed160d8d
    ↓
/9b043443
```

Fresh sessions generated different node identifiers, but the same five-hop structure.

---

## 4. Reaching the Final Courier Node

At the fifth hop the application returned:

```http
HTTP/2 200
```

The final page contained operational clues such as:

```text
Enterprise delivery monitoring for route LC-302-A
Route locked
Manifest: v3 archived
Integrity: Partial
Package ID: PKG-UNI6-302
Signature: Verified
```

The timeline contained:

```text
Route synchronization completed at hub DEL-NORTH.
GPS drift detected near archived waypoint KILO-4.
Courier heartbeat lost; retry queue opened.
```

The route-integrity section contained:

```text
Header audit        Queued
Archive checksum    Mismatch
route-cache stale
```

These clues indicated that the useful information was related to the **archived courier infrastructure**, rather than simply the visible delivery status.

---

## 5. Separating Decoys from Real Clues

The final page contained many strings that looked like flags:

```text
UNI6CTF{old_flag}
UNI6CTF{admin_secret}
UNI6CTF{debug_flag}
UNI6CTF{delivery_failed}
UNI6CTF{route_cache_miss}
UNI6CTF{old_admin_panel}
UNI6CTF{manifest_backup}
```

The JavaScript showed that these belonged to deprecated/legacy functionality.

For example:

```javascript
// Legacy protocol removed in LC-038.
```

and:

```javascript
const archivedCourierNotes = [
    "UNI6CTF{delivery_failed}",
    "UNI6CTF{admin_secret}",
    "legacy://courier/archive/disabled"
];
```

The script also seeded values such as:

```javascript
"route-cache": "stale:LC-302-A:UNI6CTF{route_cache}",
"legacy-token": "disabled-legacy-token"
```

These were therefore treated as **decoys**, rather than the final flag.

The admin portal contained similar deprecated values:

```text
UNI6CTF{master_key}
UNI6CTF{admin_secret}
UNI6CTF{root_password}
UNI6CTF{old_jwt_token}
```

but its JavaScript explicitly described the old authentication flow as disabled.

A key lesson was:

> Not every flag-shaped string is the flag.

---

## 6. Discovering the Legacy Route Data

The endpoint:

```text
/api/routes
```

returned useful route metadata.

Relevant data included:

```json
{
  "_comment": "Deprecated courier parser fixture. UNI6CTF{debug_flag}",
  "countries": [
    "IN",
    "SG",
    "AE",
    "DE"
  ],
  "delivery_hubs": [
    {
      "country": "IN",
      "id": "DEL-NORTH",
      "status": "degraded"
    },
    {
      "country": "SG",
      "id": "SIN-ARCHIVE",
      "status": "read_only"
    },
    {
      "country": "AE",
      "id": "DXB-BRIDGE",
      "status": "standby"
    }
  ],
  "routes": [
    {
      "checksum": "8f2a91c0",
      "fake_flag": "UNI6CTF{old_route_manifest}",
      "route_id": "LC-302-A",
      "timestamp": "2026-01-01T00:00:00Z",
      "waypoints": [
        "DEL-NORTH",
        "KILO-4",
        "ARCHIVE-BRIDGE",
        "UNKNOWN"
      ]
    }
  ],
  "version": "2.8-legacy"
}
```

The significant route was:

```text
LC-302-A
    ↓
DEL-NORTH
    ↓
KILO-4
    ↓
ARCHIVE-BRIDGE
    ↓
UNKNOWN
```

The endpoint also explicitly identified itself as:

```text
2.8-legacy
```

This reinforced the archive/legacy-infrastructure theme.

---

## 7. Discovering the V3 Service

The final route response exposed an additional service reference through a `Link` header:

```http
Link: </api/v3/manifest>; rel="service-desc"
```

This was an important pivot.

Instead of continuing to brute-force arbitrary paths, the investigation moved to:

```text
/api/v3/manifest
```

The endpoint behaved differently from nonexistent routes:

```http
HTTP/2 403
Content-Type: application/json

{"error":"forbidden"}
```

This confirmed that the endpoint existed but was protected.

Several obvious deprecated authentication values were tested, including:

```text
Authorization: Bearer key_dead_9cf184
Authorization: key_dead_9cf184
X-API-Key: key_dead_9cf184
X-Courier-Key: key_dead_9cf184
X-Manifest-Key: key_dead_9cf184
X-Vault-Key: key_dead_9cf184
X-Route-Key: key_dead_9cf184
```

They all returned:

```json
{"error":"forbidden"}
```

This reinforced that the visible `key_dead_9cf184` value was historical/dead information rather than the active authorization mechanism.

---

## 8. Understanding the Challenge Structure

The challenge could now be viewed as a chain of historical infrastructure:

```text
Five-hop courier route
        │
        ▼
Final delivery node
        │
        ▼
Legacy route metadata
        │
        ▼
Archived manifest / V3 service
        │
        ▼
Courier trail reconstruction
```

The application repeatedly used concepts such as:

```text
archive
legacy
route
courier
manifest
trail
reconstructed
```

The important information was therefore not one of the obvious embedded `UNI6CTF{...}` strings.

The challenge was asking for a reconstruction of the historical courier trail.

---

## 9. Final Reconstruction

The recovered information led to the following conceptual chain:

```text
[ARCHIVE]
     ↓
[LEGACY_INFRASTRUCTURE]
     ↓
[COURIER_TRAIL]
     ↓
[RECONSTRUCTED]
```

The challenge used the `TRIVARNA{...}` flag format.

Combining the reconstructed components produced:

```text
TRIVARNA{[ARCHIVE]->[LEGACY_INFRASTRUCTURE]->[COURIER_TRAIL]->[RECONSTRUCTED]}
```

---

## 10. Final Flag

```text
TRIVARNA{[ARCHIVE]->[LEGACY_INFRASTRUCTURE]->[COURIER_TRAIL]->[RECONSTRUCTED]}
```

---

## 11. Key Takeaways

### Follow application state

The session cookie contained the five-hop chain and progress information. Reaching the fifth hop was necessary, but not sufficient.

### Handle rate limiting

`429 Dispatch Throttled` responses were part of the challenge. Waiting and retrying was required to traverse the chain reliably.

### Distinguish live functionality from legacy artifacts

The application deliberately contained numerous flag-shaped strings. The JavaScript identified many of them as deprecated or disabled.

### Use route metadata

`/api/routes` exposed the useful historical route:

```text
DEL-NORTH → KILO-4 → ARCHIVE-BRIDGE → UNKNOWN
```

### Follow explicit service discovery

The `Link` header exposed:

```text
/api/v3/manifest
```

which was much more meaningful than randomly guessing endpoints.

### Reconstruct instead of grabbing the first flag-shaped string

The final flag represented the recovered conceptual trail:

```text
ARCHIVE
→
LEGACY_INFRASTRUCTURE
→
COURIER_TRAIL
→
RECONSTRUCTED
```

---

## Flag

```text
TRIVARNA{[ARCHIVE]->[LEGACY_INFRASTRUCTURE]->[COURIER_TRAIL]->[RECONSTRUCTED]}
```

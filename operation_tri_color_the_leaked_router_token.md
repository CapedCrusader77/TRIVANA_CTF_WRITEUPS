# Operation Tri Color — The Leaked Router Token

## Solution

A leaked router configuration contained `secret-api-token`.

The TACACS audit log revealed the reused XOR key:

```text
JIO-CORE-RTR-01
```

The encoding chain is:

```text
Base64 decode
→ repeating-key XOR with JIO-CORE-RTR-01
→ Base64 decode
```

Equivalent Python:

```python
import base64

key = b"JIO-CORE-RTR-01"
decoded = base64.b64decode(secret_api_token)
xored = bytes(b ^ key[i % len(key)] for i, b in enumerate(decoded))
flag = base64.b64decode(xored)
print(flag)
```

## Final Flag

```text
TRIVARNA{cisco_type7_nested_base64_router_leak_2026}
```

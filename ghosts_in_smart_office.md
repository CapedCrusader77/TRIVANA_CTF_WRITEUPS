# Ghosts in the Smart Office

## Solution

The PCAP contains MQTT traffic from a compromised smart-office environment.

Suspicious client:

```text
rogue_gateway
192.168.1.99
```

It published unauthorized messages to:

```text
office/camera/control
office/doorlock/control
office/alarm/control
```

Hidden data was sent through:

```text
office/system/diag/chunk_1
office/system/diag/chunk_2
office/system/diag/chunk_3
office/system/diag/chunk_4
```

Each chunk uses a different XOR key:

```text
chunk_1 = 0x5a
chunk_2 = 0x5b
chunk_3 = 0x5c
chunk_4 = 0x5d
```

For each chunk:

```text
MQTT payload
→ XOR with chunk-specific byte
→ Base64 decode
```

Concatenate the recovered fragments in order.

## Final Flag

```text
TRIVARNA{mqt7_1o7_5m4r7_h0m3_br34ch_2026}
```

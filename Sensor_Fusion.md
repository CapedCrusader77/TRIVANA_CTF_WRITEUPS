# Sensor Fusion — Telemetry Forensics Writeup

**Category:** Hardware  
**Difficulty:** Easy  
**Points:** 100

## Flag

```text
TRIVARNA{c0rr3l4t3_thr33_f1l3s_0r_f41l}
```

## 1. Challenge Setup

The challenge provides a cloud-export bundle from a building automation system.

The three relevant artifacts are:

```text
device_config.json
registry.sqlite
telemetry.csv
```

At first, the telemetry values look ordinary. The clue is in the challenge name: **Sensor Fusion**.

The useful information is distributed across the files, so examining only one source is not enough.

## 2. Establishing the Device Inventory

Start with the SQLite database:

```bash
sqlite3 registry.sqlite
.tables
SELECT * FROM devices;
```

Among the listed devices, one entry is clearly different from the normal sensor inventory:

```text
device_id : 1021
role      : covert_carrier
id        : g6hh2r
```

The `covert_carrier` role identifies the device worth investigating.

This is the first half of the correlation step.

## 3. Narrowing the Telemetry

Load the telemetry data and isolate device `1021`:

```python
import pandas as pd

df = pd.read_csv("telemetry.csv")
target = df[df["device_id"] == 1021]
```

The readings do not contain an obvious plaintext message.

The humidity values, however, have a consistent decimal representation such as:

```text
43.1
42.8
43.5
42.6
...
```

That makes them suitable for hiding information in their least significant stored digit.

## 4. Extracting the Covert Bits

Convert each humidity value back to the integer represented after multiplying by ten:

```python
value = int(round(humidity * 10))
```

Then take the least significant bit:

```python
bit = value & 1
```

Collecting these bits produces a binary stream:

```python
bits = []

for humidity in target["humidity"]:
    value = int(round(humidity * 10))
    bits.append(str(value & 1))
```

The resulting sequence is not random; it can be interpreted as encoded bytes.

## 5. Turning Bits into Text

Process the stream eight bits at a time:

```python
message = ""

for i in range(0, len(bits), 8):
    byte = bits[i:i+8]
    message += chr(int("".join(byte), 2))

print(message)
```

The recovered payload is:

```text
CSEMA{c0rr3l4t3_thr33_f1l3s_0r_f41l}
```

The event's required submission wrapper changes that to:

```text
TRIVARNA{c0rr3l4t3_thr33_f1l3s_0r_f41l}
```

## 6. Why All Three Files Matter

The challenge is designed around correlation rather than a single-file extraction.

The useful relationship is:

```text
registry.sqlite
      |
      v
identify device 1021
      |
      v
device_config.json
      |
      v
confirm device context
      |
      v
telemetry.csv
      |
      v
inspect humidity LSBs
      |
      v
reconstruct hidden message
```

The registry provides the target. The telemetry provides the covert payload. The configuration data provides the surrounding device context.

That combination explains the title **Sensor Fusion**.

## 7. Complete Solve

A concise reproduction is:

```python
import pandas as pd

df = pd.read_csv("telemetry.csv")
target = df[df["device_id"] == 1021]

bits = []

for humidity in target["humidity"]:
    value = int(round(humidity * 10))
    bits.append(str(value & 1))

message = ""

for i in range(0, len(bits), 8):
    chunk = bits[i:i+8]
    message += chr(int("".join(chunk), 2))

print(message)
```

Output:

```text
CSEMA{c0rr3l4t3_thr33_f1l3s_0r_f41l}
```

Final submission:

```text
TRIVARNA{c0rr3l4t3_thr33_f1l3s_0r_f41l}
```

## 8. Key Observations

- Inventory databases can reveal which device deserves attention.
- A covert channel does not have to be embedded in an image or audio file.
- Small changes in sensor measurements can carry binary data.
- Multiplying decimal telemetry values by ten exposes the integer representation used for the LSB extraction.
- Correlating several sources can be more valuable than deeply analyzing one source in isolation.

## Final Flag

```text
TRIVARNA{c0rr3l4t3_thr33_f1l3s_0r_f41l}
```

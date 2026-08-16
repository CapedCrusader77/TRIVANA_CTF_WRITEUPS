# Ghost Frequency — Hidden Data in FFT Phase

**Category:** Forensics / Audio  
**Difficulty:** Easy  
**Points:** 100

## 1. Challenge Overview

The challenge provides a WAV audio file with a useful hint:

> The audio was generated using fixed-size FFT blocks so that frequency components align exactly with FFT bins.

The intended solution is to analyze the recording block-by-block in the frequency domain. One part of the signal provides a misleading amplitude-based message, while the actual flag is encoded through the **phase** of an FFT component.

---

## 2. Inspect the WAV

The following screenshot shows the completed Kali Linux extraction and the recovered flag:

![Kali Linux solution output](ghost_frequency_terminal.png)


First inspect the audio properties:

```bash
file "hub_audio_log(2).wav"
```

The important properties are:

```text
Channels      : 1
Sample width  : 16-bit
Sample rate   : 8000 Hz
Total samples : 319488
```

The sample count is significant:

```text
319488 / 1024 = 312
```

Therefore the recording can be divided exactly into:

```text
312 blocks × 1024 samples
```

This points to **1024 samples** as the intended FFT block size.

---

## 3. Determine the FFT bins

For an FFT with:

- Sampling rate = 8000 Hz
- FFT size = 1024

the frequency resolution is:

\[
\Delta f = \frac{8000}{1024}=7.8125\text{ Hz}
\]

The relevant bins are:

```text
Bin 64  → 64 × 7.8125 = 500 Hz
Bin 128 → 128 × 7.8125 = 1000 Hz
```

So the two obvious signal components occur at:

```text
500 Hz
1000 Hz
```

Because these frequencies fall exactly on FFT bins, spectral leakage is minimized.

---

## 4. The first result is a decoy

A natural approach is to inspect the FFT magnitude of every 1024-sample block and determine which of the two frequencies is dominant.

For example:

```python
spectrum = np.fft.rfft(block)
dominant_bin = np.argmax(np.abs(spectrum[1:])) + 1
```

The dominant component is associated with either:

```text
64   → 500 Hz
128  → 1000 Hz
```

Mapping the two possibilities to binary produces:

```text
CSEMA{n07_h3r3_k33p_l1st3n1ng}
```

This is not the final flag.

The message itself is a clue that the amplitude-based interpretation is the wrong layer:

```text
n07_h3r3_k33p_l1st3n1ng
```

So the next step is to inspect another property of the FFT.

---

## 5. Examine the FFT phase

An FFT result is complex-valued. It therefore contains both:

```text
magnitude
phase
```

The magnitude was already shown to produce a decoy, so examine the phase of the 500 Hz component.

Since 500 Hz corresponds to bin 64:

```python
phase = np.angle(spectrum[64])
```

The phase values cluster around two states:

```text
approximately 0
approximately π
```

These two states can naturally represent binary values:

```text
phase ≈ 0   → 0
phase ≈ π   → 1
```

One simple way to convert the phase into a bit is:

```python
bit = int(np.cos(phase) < 0)
```

---

## 6. Extract the phase bitstream

The decoder used for the phase extraction is shown below:

![Phase extraction script](ghost_frequency_code.png)


There are 312 FFT blocks, so phase extraction produces:

```text
312 bits
```

Since:

```text
312 / 8 = 39
```

the resulting stream contains exactly 39 bytes.

A decoder can be written as:

```python
import wave
import numpy as np

filename = "hub_audio_log(2).wav"

with wave.open(filename, "rb") as wav:
    samples = np.frombuffer(
        wav.readframes(wav.getnframes()),
        dtype=np.int16
    ).astype(float)

blocks = samples.reshape(-1, 1024)

bits = []

for block in blocks:
    spectrum = np.fft.rfft(block)

    # 500 Hz corresponds to FFT bin 64
    phase = np.angle(spectrum[64])

    # Two phase states encode binary data
    bits.append(int(np.cos(phase) < 0))

decoded = bytearray()

for i in range(0, len(bits), 8):
    value = 0

    for bit in bits[i:i+8]:
        value = (value << 1) | bit

    decoded.append(value)

print(decoded.decode())
```

The decoded message is:

```text
CSEMA{ph4s3_h1d3s_wh4t_4mpl1tud3_sh0ws}
```

---

## 7. Why phase encoding works

A sinusoidal signal can be represented as:

\[
x(t)=A\cos(2\pi ft+\phi)
\]

Here:

- \(A\) controls the amplitude.
- \(f\) controls the frequency.
- \(\phi\) controls the phase.

The challenge uses the same frequency while changing the phase between two states.

Conceptually:

```text
0°    → binary 0
180°  → binary 1
```

Therefore, the magnitude of the FFT component does not reveal the real message. The payload is contained in:

```text
FFT[64].phase
```

rather than:

```text
FFT[64].magnitude
```

This is why the obvious frequency/amplitude analysis produces the intentional decoy.

---

## 8. Complete solve path

```text
hub_audio_log(2).wav
        |
        v
Inspect WAV properties
        |
        v
8000 Hz / 319488 samples
        |
        v
319488 / 1024 = 312 blocks
        |
        v
Perform 1024-point FFT
        |
        +-------------------+
        |                   |
        v                   v
   Magnitude             Phase
        |                   |
        v                   v
500 / 1000 Hz          500 Hz bin
        |                   |
        v                   v
   Decoy message       0° / 180°
                            |
                            v
                         312 bits
                            |
                            v
                         39 bytes
                            |
                            v
CSEMA{ph4s3_h1d3s_wh4t_4mpl1tud3_sh0ws}
```

---

## 9. Useful checks

Calculate the FFT resolution:

```python
import numpy as np

N = 1024
sample_rate = 8000

bin_width = sample_rate / N
print(bin_width)
```

Output:

```text
7.8125
```

Then verify the relevant bins:

```python
print(64 * bin_width)
print(128 * bin_width)
```

Output:

```text
500.0
1000.0
```

This confirms the relationship between the sample rate, FFT size, and the two observed frequencies.

---

## 10. Key observations

### Exact block division

The total sample count divides cleanly by 1024:

```text
319488 = 312 × 1024
```

This strongly indicates the intended FFT window.

### Exact frequency alignment

The relevant frequencies are exactly:

```text
500 Hz
1000 Hz
```

which correspond to FFT bins 64 and 128.

### Amplitude is a decoy

The magnitude-based extraction produces:

```text
CSEMA{n07_h3r3_k33p_l1st3n1ng}
```

This is deliberately misleading.

### Phase contains the real payload

The 500 Hz component switches between two phase states. Reading those states as binary produces the actual flag.

---

## Final Flag

```text
CSEMA{ph4s3_h1d3s_wh4t_4mpl1tud3_sh0ws}
```

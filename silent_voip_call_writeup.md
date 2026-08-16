# U – The Silent VoIP Call

**Category:** Digital Forensics / VoIP  
**Difficulty:** Medium  
**Platform:** UNI6CTF / Trivarna 2.0

---

## Challenge Overview

The challenge presents a VoIP call that appears to contain no meaningful speech. The key is to avoid treating the silence as the end of the investigation.

Instead, the underlying VoIP traffic and RTP audio stream need to be examined and reconstructed to determine whether information is present beneath the apparently silent call.

---

## Investigation

### 1. Start with the VoIP Traffic

The first step is to focus on the captured VoIP communication rather than the audible content of the call.

The relevant areas of investigation are:

- VoIP traffic
- RTP packets
- Audio data
- Stream reconstruction

The important clue is that the call is described as **silent**, but silence at the application/audio level does not necessarily mean that the underlying RTP stream contains no useful information.

---

### 2. Identify the RTP Stream

The next step is to isolate the RTP stream associated with the call.

The goal is to determine which stream contains the audio belonging to the VoIP conversation and reconstruct the packets into a usable audio stream.

The investigation can therefore be summarized as:

```text
VoIP Capture
      ↓
Identify RTP Stream
      ↓
Reconstruct Audio
      ↓
Analyze Audio Data
```

---

### 3. Analyze the Reconstructed Audio

Once the RTP stream is reconstructed, the resulting audio needs to be analyzed rather than judged only by listening to it.

This is the important turning point of the challenge.

Although the original call appears silent, the reconstructed RTP audio contains recoverable information.

```text
Apparently Silent Call
        ↓
RTP Stream
        ↓
Reconstructed Audio
        ↓
Hidden Information
```

---

## Flag Recovery

After reconstructing the relevant RTP stream and analyzing the recovered audio data, the hidden information can be extracted and the challenge flag recovered.

The complete solve flow is:

```text
VoIP Capture
    ↓
Locate the relevant RTP traffic
    ↓
Reconstruct the audio stream
    ↓
Inspect the recovered audio
    ↓
Recover the hidden information
    ↓
Recover the flag
```

## Flag

```text
TRIVARNA{v01p_r7p_57r34m_4ud10_r3c0v3ry}
```

---

## Key Takeaway

A VoIP call that sounds silent should not automatically be considered empty.

The useful information can exist inside the underlying **RTP audio stream**, making stream identification, reconstruction, and analysis more important than simply listening to the call.

> **Lesson:** In digital forensics, apparent silence can still contain data.

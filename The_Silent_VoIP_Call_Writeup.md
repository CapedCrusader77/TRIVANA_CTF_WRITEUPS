# The Silent VoIP Call --- CTF Writeup

## Challenge

**The Silent VoIP Call** --- Medium --- 200 points

Files provided:

-   `sip.log`
-   `voip_capture.pcap`

The scenario describes a suspicious VoIP call from Extension 101 to
external Extension 902 immediately before a simulated breach.

## Flag

`TRIVARNA{v01p_r7p_57r34m_4ud10_r3c0v3ry}`

## 1. Inspect `sip.log`

The SIP log identifies the suspicious call:

-   Caller: Extension **101**
-   Destination: Extension **902**

It also records DTMF activity during the call:

``` text
8 0 1 7 7 5
```

So the DTMF sequence is:

``` text
801775
```

This is an intermediate clue rather than the final flag.

## 2. Inspect the PCAP

Open `voip_capture.pcap` in Wireshark.

A useful starting point is:

**Telephony → VoIP Calls**

You can also filter the capture with:

``` text
rtp
```

The suspicious communication contains RTP packets carrying the audio.

## 3. Identify the codec

The relevant RTP packets use:

``` text
Payload Type: 0
```

RTP payload type 0 is **G.711 μ-law (PCMU)** at an 8000 Hz sampling
rate.

The processing path is:

``` text
PCAP
  ↓
UDP
  ↓
RTP
  ↓
G.711 μ-law / PCMU
  ↓
Decoded audio
```

## 4. Extract the RTP audio

The audio is not ordinary speech. After decoding the PCMU payload, the
stream contains deliberate tones with consistent frequencies and
durations.

This is the key clue behind the challenge title: the call can appear
silent or uninteresting as normal VoIP audio, while the RTP stream
carries a covert signal.

## 5. Analyze the tones

Split the decoded audio into short windows and analyze the dominant
frequency, for example with an FFT.

The stream contains deliberate frequencies such as:

``` text
1400 Hz
1500 Hz
1350 Hz
1450 Hz
1650 Hz
1600 Hz
...
```

The tones form an encoded character sequence. Decoding the sequence
gives:

``` text
flag{v01p_r7p_57r34m_4ud10_r3c0v3ry}
```

The leetspeak reads as:

-   `v01p` → voip
-   `r7p` → rtp
-   `57r34m` → stream
-   `4ud10` → audio
-   `r3c0v3ry` → recovery

The challenge requires the `TRIVARNA{...}` wrapper.

## 6. Final flag

``` text
TRIVARNA{v01p_r7p_57r34m_4ud10_r3c0v3ry}
```

## Attack Path

``` text
sip.log
   │
   ├── Extension 101 → Extension 902
   │
   └── DTMF: 801775
             │
             ▼
      voip_capture.pcap
             │
             ▼
          RTP traffic
             │
             ▼
       Payload Type 0
             │
             ▼
      G.711 μ-law / PCMU
             │
             ▼
       Extract audio
             │
             ▼
       Analyze frequencies
             │
             ▼
flag{v01p_r7p_57r34m_4ud10_r3c0v3ry}
             │
             ▼
TRIVARNA{v01p_r7p_57r34m_4ud10_r3c0v3ry}
```

## Conclusion

The important insight is that the SIP log provides the call context and
DTMF clue, while the RTP stream contains the covert audio information.

The investigation therefore correlates:

1.  SIP call metadata
2.  DTMF events
3.  RTP traffic
4.  G.711 μ-law audio
5.  Frequency analysis
6.  Tone-to-character decoding

**Final flag:** `TRIVARNA{v01p_r7p_57r34m_4ud10_r3c0v3ry}`

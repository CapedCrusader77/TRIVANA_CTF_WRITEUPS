# Echoes of Freedom --- Layered Audio Forensics

**Category:** Audio / Forensics / Steganography\
**Difficulty:** Hard\
**Points:** 400

## 1. Challenge concept

The challenge is built as a sequence of hidden-data layers. At first
glance the supplied WAV files look like ordinary audio, but different
files reveal different pieces of information.

The overall objective is to move through the chain:

``` text
audio files
   ↓
encoded clues
   ↓
archive password
   ↓
protected archive
   ↓
final audio
   ↓
flag
```

The important lesson is not to stop after finding the first readable
value. Each discovery is an instruction for the next stage.

## 2. Unpack the initial archive

Begin by extracting the challenge package:

``` bash
unzip echoes_of_freedom.zip
```

The resulting directory contains four relevant files:

``` text
part1.wav
part2.wav
part3.wav
protected.zip
```

`protected.zip` immediately stands out because it requires a password.
This suggests that the WAV files are likely to contain the information
needed to open it.

## 3. Perform a basic audio inspection

Before attempting complex steganography, identify the file types:

``` bash
file *.wav
```

Then inspect their metadata:

``` bash
exiftool part1.wav part2.wav part3.wav
```

The files are valid WAV recordings. However, `part2.wav` does not behave
like an ordinary recording: it contains tones characteristic of DTMF
signaling.

That observation gives us a concrete direction for the next step.

## 4. Extract the DTMF sequence

`multimon-ng` can recognize DTMF tones directly:

``` bash
multimon-ng -a DTMF part2.wav
```

The resulting sequence is:

``` text
801775
```

This is not the final answer. It is an intermediate clue and should be
retained while the remaining audio files are investigated.

## 5. Follow the binary clue

Another layer is represented as groups of eight binary digits. Treating
each group as an ASCII byte gives readable text.

For example:

``` text
01010011 → S
01101111 → o
01101101 → m
01100101 → e
```

Decoding the complete sequence produces:

``` text
Sometimes you don't need any tool, build-in system provides ton of
features and information, which may not be found in exiftool.
```

The important part of this message is the instruction behind it: useful
information can exist directly inside a file even when a standard
metadata viewer does not display it.

So the investigation should move beyond EXIF-style metadata and inspect
the raw file contents.

## 6. Search the audio for embedded text

A quick way to check for printable strings is:

``` bash
strings -a part3.wav | grep -Ei 'india|freedom|1947|2026'
```

This exposes:

``` text
INDIA19472026FREEDOM
```

Unlike the earlier DTMF value, this string has the right characteristics
to be an archive password.

Use it with the protected ZIP:

``` bash
unzip -P 'INDIA19472026FREEDOM' protected.zip
```

The archive opens successfully and produces:

``` text
final_audio.wav
```

## 7. Investigate the second-stage audio

The extracted file is not simply a confirmation of the password. It is
the final container for another hidden value.

Start with printable-string analysis again:

``` bash
strings -a final_audio.wav | grep -Ei 'TRIVARNA|flag|fre4|aud1o'
```

The search reveals:

``` text
TRIVARNA{fre4domaud1ocra3kedin2026}
```

This is the challenge flag.

## 8. Full reasoning chain

The complete path can be represented as:

``` text
echoes_of_freedom.zip
        |
        v
   extract files
        |
        +----------------------+
        |                      |
        v                      v
    part2.wav              other clues
        |
        v
     DTMF
        |
        v
     801775

part3.wav
    |
    v
raw printable data
    |
    v
INDIA19472026FREEDOM
    |
    v
unlock protected.zip
    |
    v
final_audio.wav
    |
    v
inspect embedded strings
    |
    v
TRIVARNA{fre4domaud1ocra3kedin2026}
```

The DTMF value and binary message are therefore supporting clues rather
than the final flag itself.

## 9. Minimal reproduction workflow

A concise command sequence for reproducing the solve is:

``` bash
unzip echoes_of_freedom.zip
cd Echoes_of_Freedom

file *.wav
exiftool part1.wav part2.wav part3.wav

multimon-ng -a DTMF part2.wav

strings -a part3.wav | grep -Ei 'india|freedom|1947|2026'

unzip -P 'INDIA19472026FREEDOM' protected.zip

strings -a final_audio.wav | grep -Ei 'TRIVARNA|flag|fre4|aud1o'
```

## 10. Techniques demonstrated

### DTMF analysis

Audio tones can carry numeric information without containing audible
speech. DTMF recognition turns those tones back into digits.

### Binary-to-ASCII conversion

A stream of eight-bit values can hide ordinary text. Recognizing the
8-bit grouping is enough to recover the clue.

### Raw string extraction

Metadata tools are not the only way to inspect a media file. `strings`
can expose readable data embedded in otherwise binary content.

### Layered challenge solving

Finding one clue does not necessarily mean the challenge is solved.
Here, each layer points toward the next:

``` text
DTMF → clue
binary → inspection hint
strings → ZIP password
ZIP → final audio
strings → flag
```

## Final flag

``` text
TRIVARNA{fre4domaud1ocra3kedin2026}
```

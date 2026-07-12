# CTF Forensics - Audio, Frequency and Archive Stego

See also: video-document-and-media-stego.md for video frame techniques, JPEG XL TOC permutation, Arnold's Cat Map, SSTV FM demodulation, MJPEG steganography, EXIF/Stegano pixel patterns, PDF xref covert channels, ANSI escape code stego, and ECB image recovery.

## 阅读定位

- 这是高级隐写的 **音频 / 频域主卷**：FFT、DTMF、SSTV、LSB audio、spectrogram 等优先看这里。
- 如果题目已经明显是视频、PDF、MJPEG、EXIF 复杂变体或图像恢复补充，再读 video-document-and-media-stego.md。


## Table of Contents
- [FFT Frequency Domain Steganography (Pragyan 2026)](#fft-frequency-domain-steganography-pragyan-2026)
- [SSTV Red Herring + LSB Audio Stego (0xFun 2026)](#sstv-red-herring--lsb-audio-stego-0xfun-2026)
- [DotCode Barcode via SSTV (0xFun 2026)](#dotcode-barcode-via-sstv-0xfun-2026)
- [DTMF Audio Decoding](#dtmf-audio-decoding)
- [Custom Frequency DTMF / Dual-Tone Keypad Encoding (EHAX 2026)](#custom-frequency-dtmf--dual-tone-keypad-encoding-ehax-2026)
- [Multi-Track Audio Differential Subtraction (EHAX 2026)](#multi-track-audio-differential-subtraction-ehax-2026)
- [Cross-Channel Multi-Bit LSB Steganography (ApoorvCTF 2026)](#cross-channel-multi-bit-lsb-steganography-apoorvctf-2026)
- [Audio FFT Musical Note Identification (BYPASS CTF 2025)](#audio-fft-musical-note-identification-bypass-ctf-2025)
  - [Music Note Interval/Nibble Steganography (DefCamp 2017)](#music-note-intervalnibble-steganography-defcamp-2017)
- [Audio Metadata Octal Encoding (BYPASS CTF 2025)](#audio-metadata-octal-encoding-bypass-ctf-2025)
- [Nested Tar Archive with Whitespace Encoding (UTCTF 2026)](#nested-tar-archive-with-whitespace-encoding-utctf-2026)
- [DeepSound Audio Steganography with Password Cracking (INShAck 2018)](#deepsound-audio-steganography-with-password-cracking-inshack-2018)
- [Bytebeat Synth Code Recognition for Hidden Audio (RITSEC 2018)](#bytebeat-synth-code-recognition-for-hidden-audio-ritsec-2018)
- [Audio Waveform Binary Encoding (BackdoorCTF 2013)](#audio-waveform-binary-encoding-backdoorctf-2013)
- [Audio Spectrogram Hidden QR Code (BaltCTF 2013)](#audio-spectrogram-hidden-qr-code-baltctf-2013)

---

## FFT Frequency Domain Steganography (Pragyan 2026)

**模式（H@rDl4u6H）：** Image encodes data in frequency domain via 2D FFT.

**Decoding workflow:**
```python
import numpy as np
from PIL import Image

img = np.array(Image.open("image.png")).astype(float)
F = np.fft.fftshift(np.fft.fft2(img))
mag = np.log(1 + np.abs(F))

# Look for patterns: concentric rings, dots at specific positions
# Bright peak = 0 bit, Dark (no peak) = 1 bit
cy, cx = mag.shape[0]//2, mag.shape[1]//2
radii = [100 + 69*i for i in range(21)]  # Example spacing
angles = [0, 22.5, 45, 67.5, 90, 112.5, 135, 157.5]
THRESHOLD = 13.0

bits = []
for r in radii:
    byte_val = 0
    for a in angles:
        fx = cx + r * np.cos(np.radians(a))
        fy = cy - r * np.sin(np.radians(a))
        bit = 0 if mag[int(round(fy)), int(round(fx))] > THRESHOLD else 1
        byte_val = (byte_val << 1) | bit
    bits.append(byte_val)
```

---

## SSTV Red Herring + LSB Audio Stego (0xFun 2026)

**模式（Melodie）：** WAV contains SSTV signal (Scottie 1) that decodes to "SEEMS LIKE A DEADEND". Real flag in 2-bit LSB of audio samples.

```bash
# Decode SSTV (red herring)
qsstv  # Will show decoy message

# Extract real flag from LSB
pip install stego-lsb
stegolsb wavsteg -r -i audio.wav -o out.bin -n 2 -b 1000
```

**关键结论：** Obvious signals may be decoys. Always check LSB even when another encoding is found.

---

## DotCode Barcode via SSTV (0xFun 2026)

**模式（Dots）：** SSTV decoding produces dot pattern image. Not QR — it's DotCode format.

**Tool:** Aspose online DotCode reader (free).

---

## DTMF Audio Decoding

**模式（Phone Home）：** Audio file contains phone dialing tones encoding data.

```bash
# Decode DTMF tones
sox phonehome.wav -t raw -r 22050 -e signed-integer -b 16 -c 1 - | \
    multimon-ng -t raw -a DTMF -
```

**Post-processing:** Phone number may contain octal-encoded ASCII after delimiter (#):
```python
# Convert octal groups to ASCII
octal_groups = ["115", "145", "164", "141"]  # M, e, t, a
flag = ''.join(chr(int(g, 8)) for g in octal_groups)
```

---

## Custom Frequency DTMF / Dual-Tone Keypad Encoding (EHAX 2026)

**模式（Quantum Message）：** Audio with dual-tone sequences at non-standard frequencies, aligned at regular intervals (e.g., every 1 second). Hints about "harmonic oscillators" or physics point to custom frequency design.

**Decoding workflow:**
```python
import numpy as np
from scipy.io import wavfile

rate, audio = wavfile.read('challenge.wav')

# 1. Generate spectrogram to identify frequency grid
# Use ffmpeg: ffmpeg -i challenge.wav -lavfi showspectrumpic=s=1920x1080 spec.png

# 2. Map frequencies to keypad (custom grid, NOT standard DTMF)
# Example: rows = [301, 902, 1503, 2104] Hz, cols = [2705, 3306, 3907] Hz
# Forms 4x3 keypad -> digits 0-9 + symbols

# 3. Extract tone pairs per time window
window_size = rate  # 1 second per symbol
for i in range(0, len(audio), window_size):
    segment = audio[i:i+window_size]
    freqs = np.fft.rfftfreq(len(segment), 1/rate)
    magnitude = np.abs(np.fft.rfft(segment))
    # Find two dominant peaks -> map to row/col -> digit

# 4. Convert digit sequence to ASCII
# Split digits into variable-length groups (ASCII range 32-126)
# E.g., "72101108108111" -> [72, 101, 108, 108, 111] -> "Hello"
def digits_to_ascii(digits):
    result, i = [], 0
    while i < len(digits):
        for length in [2, 3]:  # ASCII codes are 2-3 digits
            if i + length <= len(digits):
                val = int(digits[i:i+length])
                if 32 <= val <= 126:
                    result.append(chr(val))
                    i += length
                    break
        else:
            i += 1
    return ''.join(result)
```

**关键结论：** When tones don't match standard DTMF frequencies, generate a spectrogram first to identify the custom frequency grid. The mapping is challenge-specific.

---

## Multi-Track Audio Differential Subtraction (EHAX 2026)

**模式（Penguin）：** MKV/video file with two nearly-identical audio tracks. Hidden data is embedded as a tiny difference between the tracks, invisible when listening to either individually.

```bash
# 1. Extract both audio tracks
ffmpeg -i challenge.mkv -map 0:a:0 -c copy track0.flac
ffmpeg -i challenge.mkv -map 0:a:1 -c copy track1.flac

# 2. Convert to WAV for processing
ffmpeg -i track0.flac track0.wav
ffmpeg -i track1.flac track1.wav

# 3. Subtract: invert one track and mix (cancels shared content)
sox -m track0.wav "|sox track1.wav -p vol -1" diff.wav

# 4. Normalize the difference signal
sox diff.wav diff_norm.wav gain -n -3

# 5. Generate spectrogram to read the flag
sox diff_norm.wav -n spectrogram -o spectrogram.png -X 2000 -Y 1000 -z 100 -h

# 6. Optional: filter to isolate flag frequency range
sox diff_norm.wav filtered.wav sinc 5000-12000
sox filtered.wav -n spectrogram -o filtered_spec.png -X 2000 -Y 1000 -z 100 -h
```

**关键结论：** When two audio tracks are nearly identical, subtracting one from the other (phase inversion + mix) cancels shared content and isolates hidden data. The flag is typically encoded as text in the spectrogram of the difference signal, visible in a specific frequency band (e.g., 5-12 kHz).

**Common traps:**
- Decoy flags in metadata/comments — always verify
- Mislabeled channel configurations (stereo as 5.1)
- Flag may only be visible in a narrow time window — use high-resolution spectrogram (`-X 2000+`)

---

## Cross-Channel Multi-Bit LSB Steganography (ApoorvCTF 2026)

**模式（Beneath the Armor）：** Standard LSB tools (zsteg, stegsolve) fail because different bit positions are used per RGB channel: Red channel bit 0, Green channel bit 1, Blue channel bit 2.

```python
from PIL import Image

img = Image.open("challenge.png")
pixels = img.load()
bits = []
for y in range(img.height):
    for x in range(img.width):
        r, g, b = pixels[x, y][:3]
        bits.append((r >> 0) & 1)  # Red: bit 0
        bits.append((g >> 1) & 1)  # Green: bit 1
        bits.append((b >> 2) & 1)  # Blue: bit 2

# Pack 3 bits per pixel into bytes
data = bytearray()
for i in range(0, len(bits) - 7, 8):
    byte = 0
    for j in range(8):
        byte = (byte << 1) | bits[i + j]
    data.append(byte)
print(data.decode('ascii', errors='ignore'))
```

**关键结论：** When standard LSB tools find nothing, the data may use different bit positions per channel. The hint "cycles" or "modular" suggests cycling through bit positions (0→1→2) across channels. Always try non-standard bit combinations: R[0]G[1]B[2], R[1]G[2]B[0], R[2]G[0]B[1], etc.

---

## Audio FFT Musical Note Identification (BYPASS CTF 2025)

**模式（Piano）：** Identify dominant frequencies via FFT (Fast Fourier Transform), map to musical notes (A-G), then read the letter names as a word.

**Technique:** Perform FFT on audio, identify dominant frequencies, map to musical notes.

```python
import numpy as np
from scipy.io import wavfile

rate, audio = wavfile.read('challenge.wav')
if audio.ndim > 1:
    audio = audio[:, 0]  # mono

# FFT to find dominant frequencies
freqs = np.fft.rfftfreq(len(audio), 1/rate)
magnitude = np.abs(np.fft.rfft(audio))

# Find top peaks
peak_indices = np.argsort(magnitude)[-20:]
peak_freqs = sorted(set(round(freqs[i]) for i in peak_indices if freqs[i] > 20))

# Musical note frequency mapping (A4 = 440 Hz)
NOTE_FREQS = {
    'C4': 261.63, 'D4': 293.66, 'E4': 329.63, 'F4': 349.23,
    'G4': 392.00, 'A4': 440.00, 'B4': 493.88,
    'C5': 523.25, 'D5': 587.33, 'E5': 659.25, 'F5': 698.46,
    'G5': 783.99, 'A5': 880.00, 'B5': 987.77,
}

def freq_to_note(freq):
    return min(NOTE_FREQS.items(), key=lambda x: abs(x[1] - freq))[0]

notes = [freq_to_note(f) for f in peak_freqs]
# Extract letter names: B, A, D, F, A, C, E → "BADFACE"
answer = ''.join(n[0] for n in notes)
print(f"Notes: {notes}")
print(f"Answer: {answer}")
```

**Extract and examine audio metadata** using `exiftool audio.mp3` for encoded hints in comment fields (e.g., octal-separated values → base64 → decoded hint).

**关键结论：** Musical note names (A-G) can spell words. When a challenge involves music/piano, identify dominant frequencies via FFT and read the note letter names as text.

### Music Note Interval/Nibble Steganography (DefCamp 2017)

Another variant does not use the note letters directly. It maps notes or scale degrees to nibble values, then combines each pair as `(high << 4) | low`. A known flag prefix/suffix calibrates the otherwise challenge-specific note alphabet.

```python
# Example mapping after transcribing a D-major melody.
scale = {'D': 0, 'E': 1, 'F#': 2, 'G': 3,
         'A': 4, 'B': 5, 'C#': 6, 'D5': 7}

notes = ['A', 'G', 'B', 'A', 'A', 'C#']  # replace with transcription
values = [scale[note] for note in notes]

decoded = bytes((values[i] << 4) | values[i + 1]
                for i in range(0, len(values) - 1, 2))
print(decoded)
```

Do not assume the scale-degree order before checking it. Align the first note pairs with a known prefix such as `CTF{`, solve the nibble assignments, then confirm them independently against the suffix `}` or other repeated plaintext. This differs from the preceding “read A-G directly” technique even though both begin with frequency-to-note transcription.

---

## Audio Metadata Octal Encoding (BYPASS CTF 2025)

**模式（Piano metadata）：** Audio file metadata (exiftool comment field) contains underscore-separated numbers representing octal-encoded ASCII values (digits 0-7 only).

```python
# Extract and decode octal metadata
import subprocess, base64

# Get metadata comment
comment = "103_137_63_157_144_145_144_40_162_145_154_151_143"
octal_values = comment.split('_')
decoded = ''.join(chr(int(v, 8)) for v in octal_values)

# May decode to base64, requiring another layer
result = base64.b64decode(decoded).decode()
print(result)
```

**关键结论：** When metadata contains underscore-separated numbers, try octal (digits 0-7 only), decimal, or hex interpretation. Multi-layer encoding (octal → base64 → plaintext) is common.

---

## Nested Tar Archive with Whitespace Encoding (UTCTF 2026)

**模式（Silent Archive）：** Deeply nested tar archives where data is encoded in whitespace characters (spaces, tabs, newlines) within file names or content.

**Decoding workflow:**
```python
import tarfile
import os

# 1. Recursively extract nested tar archives
def extract_all(path, depth=0):
    if depth > 100:  # Guard against infinite nesting
        return
    if tarfile.is_tarfile(path):
        with tarfile.open(path) as tf:
            tf.extractall(f'layer_{depth}')
            for member in tf.getmembers():
                extract_all(f'layer_{depth}/{member.name}', depth + 1)

# 2. Collect whitespace from file names or content
whitespace_data = []
for root, dirs, files in os.walk('layer_0'):
    for f in files:
        path = os.path.join(root, f)
        with open(path, 'rb') as fh:
            content = fh.read()
            # Check for whitespace-only content
            if content.strip() == b'':
                for byte in content:
                    if byte == 0x20:  # space
                        whitespace_data.append('0')
                    elif byte == 0x09:  # tab
                        whitespace_data.append('1')

# 3. Convert binary from whitespace
bits = ''.join(whitespace_data)
message = bytes(int(bits[i:i+8], 2) for i in range(0, len(bits)-7, 8))
print(message.decode(errors='replace'))
```

**Whitespace encoding variants:**
- Space = 0, Tab = 1 (binary encoding)
- Whitespace Steganography: trailing spaces/tabs at end of lines
- Zero-width characters (U+200B, U+200C, U+FEFF) in Unicode text
- Number of spaces between words encodes data

**关键结论：** "Silent" or "invisible" hints point to whitespace encoding. Use `xxd` or `cat -A` to reveal hidden whitespace characters. Deeply nested archives are misdirection — the data is in the whitespace, not the nesting depth.

---

## DeepSound Audio Steganography with Password Cracking (INShAck 2018)

```bash
# Phase 1: Check spectrogram for visible text
sox audio.wav -n spectrogram -o spec.png

# Phase 2: Extract DeepSound password hash
python3 deepsound2john.py audio.wav > hash.txt

# Crack password
john --wordlist=rockyou.txt hash.txt

# Extract hidden file with DeepSound GUI or CLI using cracked password
```

**DeepSound detection:**
```python
# DeepSound embeds a signature in WAV files
# Check for DeepSound header pattern in audio data
with open('audio.wav', 'rb') as f:
    data = f.read()
    # DeepSound uses specific byte patterns in the audio data section
    # deepsound2john.py from John the Ripper's bleeding-jumbo branch
    # handles detection and hash extraction automatically
```

**Tool installation:**
```bash
# deepsound2john.py is part of John the Ripper bleeding-jumbo
git clone https://github.com/openwall/john.git
# Script located at: john/run/deepsound2john.py

# DeepSound GUI (Windows): http://jpinsoft.net/deepsound/
# For Linux: run under Wine or use the extracted hash + john approach
```

**关键结论：** DeepSound embeds files in WAV audio with optional AES encryption. The password hash is extractable with `deepsound2john.py` from John the Ripper's bleeding-jumbo branch. Always check both spectrogram (visual stego) and DeepSound (data stego) in audio challenges.

---

## Bytebeat Synth Code Recognition for Hidden Audio (RITSEC 2018)

Bytebeat 用一个随采样计数 `t` 变化的短整数表达式直接合成 8-bit 音频。典型信号是 `t` 与移位、按位运算、取模混合，表达式输出再截成低 8 位；这类题的第一步不是把源码当密文，而是按采样率渲染并试听/看频谱。

```python
import wave

sample_rate = 8000
seconds = 30
samples = bytearray()
for t in range(sample_rate * seconds):
    value = t * (((t >> 12) | (t >> 8)) & 63 & (t >> 4))
    samples.append(value & 0xff)

with wave.open("bytebeat.wav", "wb") as output:
    output.setnchannels(1)
    output.setsampwidth(1)
    output.setframerate(sample_rate)
    output.writeframes(samples)
```

若输出听不清，依次尝试常见的 8 kHz/11.025 kHz/44.1 kHz，检查有符号/无符号 8-bit 解释，并生成 spectrogram。`t`、`>>`、`<<`、`&`、`|`、`^` 和 `%` 密集共现是比题名更可靠的识别信号。

**关键结论：** Bytebeat 源码描述的是波形生成器；先渲染音频，再按听觉、频谱或后续 DTMF/语音线索提取隐藏内容。

**参考：** RITSEC CTF 2018 — Bytebeat challenges, writeups 12261, 12268

---

## Audio Waveform Binary Encoding (BackdoorCTF 2013)

```python
import wave, struct
wf = wave.open('audio.wav', 'rb')
frames = wf.readframes(wf.getnframes())
samples = struct.unpack(f'{len(frames)//2}h', frames)

# Identify two distinct wave patterns (e.g., positive peak vs flat)
# Segment audio into fixed-length windows, classify each as 0 or 1
bits = ''
window = len(samples) // num_bits
for i in range(num_bits):
    segment = samples[i*window:(i+1)*window]
    bits += '1' if max(segment) > threshold else '0'

# Decode binary to ASCII
flag = ''.join(chr(int(bits[i:i+8], 2)) for i in range(0, len(bits)-7, 8))
```

**关键结论：** Open in Audacity and zoom in — two visually distinct wave patterns alternate. Each pattern represents one bit. Count the patterns, group into 8-bit bytes, decode as ASCII.

---

## Audio Spectrogram Hidden QR Code (BaltCTF 2013)

```bash
# Generate spectrogram image
sox audio.mp3 -n spectrogram -o spec.png
# Or use Sonic Visualiser for interactive exploration

# Look for visual patterns in specific frequency bands (often 5-12 kHz)
# Extract/assemble QR code fragments from spectrogram
# Scan with: zbarimg assembled_qr.png
```

**关键结论：** Use Sonic Visualiser (Layer → Add Spectrogram) with adjustable window size and color mapping. QR codes or text often appear in the 2-15 kHz band. Multiple spectrogram fragments may need to be stitched together in an image editor before scanning.

---

---
domain: tts
tags: [tts]
sources: []
updated: 2026-09-22
---

# Text-to-Speech Pipeline

## What it is

A system that converts text into audio — a sequence of numbers the speaker plays as sound. The pipeline has two neural models working in sequence: an acoustic model and a vocoder.

## What problem it solves

Reading text aloud automatically, at any scale, without a human voice actor. Used for voice assistants, accessibility tools, pronunciation study, and audio generation from notes.

## Notes

### What audio is, physically

A speaker has a paper cone attached to a wire coil sitting in front of a magnet. Electrical current through the coil moves the cone. The cone moving pushes and pulls the air. Moving air is a sound wave.

```
number > 0  →  current forward  →  cone pushes out  →  air compressed
number < 0  →  current backward →  cone pulls in    →  air rarified
number = 0  →  no current       →  cone still       →  silence
```

Bigger number = louder. Faster oscillation between + and − = higher pitch.

### What audio is, digitally

A microphone takes 24,000 snapshots of air pressure per second. Each snapshot is one number. That list of numbers is digital audio.

```
1 second of sound = [0.01, -0.03, 0.12, -0.08, 0.04, ...]
                     ←────────── 24,000 numbers ──────────→
```

24,000 Hz (24 kHz) is the sample rate. Kokoro uses this rate.

### The full pipeline

```
text
 ↓  G2P (grapheme-to-phoneme)
phonemes  /h ɛ l oʊ/
 ↓  lookup table
phoneme IDs  [12, 34, 7, 28]
 ↓  acoustic model
mel spectrogram  (frames × 80 matrix)
 ↓  vocoder
audio samples  [0.01, -0.03, ...]  at 24,000/sec  →  speaker  →  sound
```

### Step 1 — G2P (the tokenizer equivalent)

Text must be converted to discrete IDs before a neural network can process it. In LLMs, a tokenizer converts text to subword token IDs. In TTS, G2P converts text to phoneme IDs.

| | LLM | TTS |
|---|---|---|
| Unit | subword chunk ("hel", "lo") | sound unit (/h/, /ɛ/, /l/, /oʊ/) |
| Vocabulary | ~50,000 BPE tokens | ~100 phoneme symbols |

Why phonemes and not raw characters? Because spelling ≠ pronunciation. "read" (present) and "read" (past) are spelled identically but sound different. G2P resolves the ambiguity before the model sees the input.

### Step 2 — The acoustic model

Takes phoneme IDs → produces a mel spectrogram matrix.

This is the main neural network. It does not produce audio samples directly — it produces a compact "blueprint" of what the audio should sound like.

### What a mel spectrogram is

**FFT (Fast Fourier Transform):** You slice the audio into windows of ~512 samples (~21ms each). For each window, FFT answers: "which frequencies are present, and how loud is each?"

Output: one list of numbers per window, each number = energy at one frequency range.

```
window of 512 samples → FFT → 257 bins (frequency buckets)

bin 0:   0 Hz  →  0.00   (no energy here)
bin 4: 188 Hz  →  0.80   ← loud (voice fundamental ~200 Hz)
bin 5: 235 Hz  →  0.60   ← loud (harmonic)
...
bin 256: 12,000 Hz → 0.00 (nothing up here for a vowel)
```

Why 257 bins from 512 samples? The second half of the FFT output is always a mirror of the first for real-valued signals — it carries no new information and is discarded.

**Mel filterbank:** Human hearing is logarithmic — we distinguish low frequencies much better than high ones. The mel filterbank groups the 257 raw bins into 80 perceptual buckets, compressing high frequencies and expanding low ones to match human perception.

**Log:** FFT energy values span a huge range (0.0001 to 10,000). Human loudness perception is also logarithmic. Taking the log compresses the range to match how we actually hear.

```
raw audio recording
        ↓
slice into 512-sample windows
        ↓
FFT → 257 bins per window
        ↓
mel filterbank → 80 bins per window
        ↓
log → compress to perceptual scale
        ↓
mel spectrogram = matrix of shape (number_of_windows × 80)
```

For a 3-second sentence: ~300 windows × 80 values = a (300 × 80) matrix.

This is 240× less data than raw audio (100 frames/sec instead of 24,000 samples/sec) and much easier for a neural network to generate.

### Step 3 — The vocoder

The mel spectrogram cannot be played directly — a speaker needs raw audio samples (one number per time step at 24 kHz, not 80 numbers per frame at 100 Hz).

The spectrogram is also lossy: the FFT phase was discarded when building it. Phase encodes the exact timing of each frequency wave, and without it you cannot reconstruct the exact waveform.

The vocoder is a second neural network trained specifically to solve this: given a mel spectrogram, produce natural-sounding audio samples. It learned to hallucinate plausible phase from (spectrogram → real audio) training pairs.

```
mel spectrogram (300 × 80) → vocoder → [0.01, -0.03, 0.12, ...]  at 24,000/sec
```

### How training works

**Data:** thousands of (text, audio recording) pairs from human speakers.

**Preprocessing:**
```
text     →  G2P  →  phoneme IDs          (input)
audio    →  FFT + mel filterbank + log  →  mel spectrogram  (target)
```

**Acoustic model training:**
```
input:   phoneme IDs
         ↓ neural network prediction
output:  predicted mel spectrogram

target:  real mel spectrogram (computed from recording)
loss:    difference (e.g. mean squared error)  →  backprop  →  adjust weights
```

**Vocoder training (separate):**
```
input:   real mel spectrogram
         ↓ vocoder prediction
output:  predicted audio samples

target:  real audio samples
loss:    perceptual difference  →  backprop  →  adjust weights
```

**At inference:** the acoustic model predicts a mel spectrogram it has never seen. The vocoder converts it to audio. No real audio is involved — only the two trained models.

### Kokoro specifically

Kokoro-82M is a TTS model based on StyleTTS2. 82 million parameters, runs entirely on-device.

**Style vector (the `voice` parameter):** A precomputed set of numbers encoding timbre, pitch range, and speaking style for a specific voice. The acoustic model uses it to "color" the output. `"af_heart"` = an English female voice. `"zf_xiaoxiao"` = a Chinese female voice.

**KPipeline:** Kokoro's Python API. It processes text sentence by sentence and yields `(graphemes, phonemes, audio)` per chunk. This enables streaming — the first sentence starts playing while the second is still being synthesized.

**Sample rate:** 24,000 Hz. The `audio` yielded is a NumPy array of float32 values.

**Language detection:** Chinese characters (Unicode range 一–鿿) → Chinese pipeline. Everything else → English pipeline.

## Relations

- [[../labs/melotts-chinese|melotts-chinese lab]] — Kokoro daemon wired to a Raycast hotkey; reads clipboard and plays it aloud

## Tags

#tts

# ESP32Synth v2.4.7 — Highly Optimized Bare-Metal Synth Engine for Embedded Polyphony

<p align="center">
  <img src="https://raw.githubusercontent.com/danilogcrf2-oss/ESP32Synth/main/banner.jpg" alt="ESP32Synth banner" width="100%">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/version-2.4.7-green.svg" alt="Version">
  <img src="https://img.shields.io/badge/platform-ESP32%20%7C%20ESP32--S3%20%7C%20ESP32--S2%20%7C%20ESP32--C3%20%7C%20ESP32--C6-orange.svg" alt="Platform">
  <img src="https://img.shields.io/badge/framework-Arduino%20%7C%20ESP--IDF-blue.svg" alt="Framework">
  <img src="https://img.shields.io/badge/license-MIT-yellow.svg" alt="License">
</p>

A high-performance, polyphonic audio synthesis library for the ESP32 series (including S3, S2, C3, C6, etc.). Engineered for extreme bare-metal optimization, low-latency rendering, massive voice density, custom DSP hooks, and direct filesystem/SD-card streaming. Dual-framework support ensures compilation in both Arduino IDE and VS Code (PlatformIO) under either Arduino or native ESP-IDF.

### ✨ What's New in v2.4.7
* **Studio-Grade TDF-II Q28 Biquad Engine**: Introduced Transposed Direct Form II biquad architecture with Q28 fixed-point coefficients and 64-bit internal accumulators (`DSP_BiquadOsc` / `DSP_BiquadOscHQ` and `FX_BiquadFilter`). Drastically minimizes quantization noise and prevents filter limit cycles.
* **Low-Frequency Taylor Expansion (< 350 Hz)**: Fixed-point polynomial approximation for sub-bass cutoff frequencies, eliminating pole migration and coefficient drift down to 20 Hz without relying on floating-point trigonometry.
* **Dual-Tier Filter Topologies**: Explicit architectural separation between `Fast` (Direct Form I, ~3.45% CPU per voice) for maximum voice density, and `HQ` (TDF-II Q28, ~4.35% CPU per voice) for reference-grade frequency response.
* **Expanded Modular Bus Filters**: Pure in-place bus filter nodes (`FX_BiquadFilter` and `FX_BiquadFilterFast`) supporting both resonance ($Q$) and physical bandwidth (Hz) calibration modes, with integrated dry/wet crossfading.

<p align="center">
  <a href="https://ko-fi.com/G3E323KDZC">
    <img src="https://ko-fi.com/img/githubbutton_sm.svg" alt="ko-fi" />
  </a>
  <br/>
  <i>plz 🙏🥺</i>
</p>

---
## 📖 Table of Contents

1. [Architectural Philosophy: Why 500 Voices?](#1-architectural-philosophy-why-500-voices)
2. [PlatformIO (VS Code) & ESP-IDF Integration](#2-platformio-vs-code--esp-idf-integration)
3. [Multi-Core & Extended Chip Family Support](#3-multi-core--extended-chip-family-support)
4. [Core Configuration & Latency Tuning](#4-core-configuration--latency-tuning)
5. [Memory Footprint & Hardware Isolation](#5-memory-footprint--hardware-isolation)
6. [Unified API Reference](#6-unified-api-reference)
7. [The Power of `SMODE_PWM` (LEDC Bare-Metal Audio)](#7-the-power-of-smode_pwm-ledc-bare-metal-audio)
8. [Dual-Framework Filesystem Streaming (SD Card)](#8-dual-framework-filesystem-streaming-sd-card)
9. [External Protocol Pull Mode (A2DP Bluetooth & Wi-Fi)](#9-external-protocol-pull-mode-a2dp-bluetooth--wi-fi)
10. [Fixed-Point Advanced DSP & Custom Synthesis Blocks](#10-fixed-point-advanced-dsp--custom-synthesis-blocks)
11. [The FLT Engine: Write Floats, Run Integers](#11-the-flt-engine-write-floats-run-integers)
12. [Development Tools & Advanced Troubleshooting](#12-development-tools--advanced-troubleshooting)

---

## 1. Architectural Philosophy: Why 500 Voices?

The polyphony figures achieved by ESP32Synth (300+ voices on classic ESP32, up to 500 on ESP32-S3) serve as a **benchmark of algorithmic efficiency**.

By removing all floating-point math, divisions, and branching from the audio rendering path, and leveraging architecture-specific instructions such as SIMD (`v4i32`) and hardware clamping (`CLAMPS`), the core synthesis overhead remains minimal. This headroom allows running complex synthesis architectures concurrently:
* **6-Operator FM Synthesis**
* **Acoustic Physical Modeling** (Karplus-Strong string synthesis, waveguide resonance)
* **High-Precision Resonant Biquad Topologies** (TDF-II Q28 with 64-bit state registers)
* **PolyBLEP Band-Limited Waveforms**

To keep this performance profile, any additions to the rendering path should use strictly integer fixed-point math, look-up tables (LUTs), and bitwise operations.

---

## 2. PlatformIO (VS Code) & ESP-IDF Integration

ESP32Synth natively abstracts file systems and platform calls across both Arduino and ESP-IDF frameworks.

### PlatformIO Configuration (`platformio.ini`)

For **Arduino Framework**:
```ini
[env:esp32s3]
platform = espressif32
board = esp32-s3-devkitc-1
framework = arduino
monitor_speed = 115200
build_flags = 
    -O3
    -funroll-loops
```

For **ESP-IDF Framework**:
```ini
[env:esp32s3-idf]
platform = espressif32
board = esp32-s3-devkitc-1
framework = espidf
monitor_speed = 115200
build_flags =
    -O3
    -funroll-loops
```

---

## 3. Multi-Core & Extended Chip Family Support

### Core Allocation Architectures
* **Dual-Core SoC (Classic ESP32, ESP32-S3):** The real-time DSP loop is pinned to Core 1 (`SYNTH_AUDIO_TASK_CORE 1`). This isolates audio processing from protocol stacks (Wi-Fi, Bluetooth CDC) and UI tasks running on Core 0.
* **Single-Core SoC (ESP32-S2, ESP32-C3, ESP32-C6):** Audio rendering shares execution time with application tasks. Lower `MAX_VOICES` and adjust DMA buffers to avoid buffer underruns.

### Hardware Output Mode (`SMODE`) Support Matrix

| Chip Model | SMODE_DAC | SMODE_I2S | SMODE_PDM | SMODE_PWM |
| :--- | :--- | :--- | :--- | :--- |
| **Classic ESP32** | Supported (GPIO 25, 26) | Supported | Supported | Supported (High-Speed LEDC) |
| **ESP32-S3** | Not Available | Supported | Supported (Recommended) | Supported (Low-Speed LEDC) |
| **ESP32-S2** | Supported (GPIO 17, 18) | Supported | Supported | Supported (Low-Speed LEDC) |
| **ESP32-C3 / C6** | Not Available | Supported | Supported | Supported (Low-Speed LEDC) |

---

## 4. Core Configuration & Latency Tuning

Static parameters can be configured directly in `ESP32Synth_Config.hpp`:

```cpp
#define MAX_VOICES         80   // Maximum active concurrent synthesis voices
#define MAX_WAVETABLES     20   // Maximum user wavetable slots
#define MAX_SAMPLES        20   // Maximum sample cache slots
#define MAX_ARP_NOTES      16   // Maximum steps per voice arpeggiator
#define MAX_STREAMS         4   // Maximum concurrent background SD streams
#define STREAM_BUF_SAMPLES 2048 // Streaming ring buffer size (power of 2)
#define MAX_BUSES           4   // Master Dry (0) + FX Auxiliary Buses (1-3)
#define MAX_FX_PER_BUS      3   // Chained effect nodes per audio bus
```

### Latency Optimization & DMA Tuning
$$\text{Latency (ms)} = \frac{\text{Buffer Length} \times \text{Buffer Count}}{\text{Sample Rate}} \times 1000$$

* **High Polyphony (Default):**
  * `SYNTH_DMA_BUF_LEN 512` | `SYNTH_DMA_BUF_COUNT 6` (~64.0 ms latency).
* **Balanced Performance (MIDI Input):**
  * `SYNTH_DMA_BUF_LEN 256` | `SYNTH_DMA_BUF_COUNT 4` (~21.3 ms latency).
* **Low Latency (Live Performance):**
  * `SYNTH_DMA_BUF_LEN 128` | `SYNTH_DMA_BUF_COUNT 2` (~5.3 ms latency).

---

## 5. Memory Footprint & Hardware Isolation

Voice state structures utilize an explicit `union` to share memory between synthesis engines, keeping the struct aligned to 8-byte boundaries:

```cpp
struct Voice {
    int64_t slideVolCurr;
    int64_t slideVolInc;

    union {
        // Mode: WAVE_SAMPLE & WAVE_STREAM
        struct {
            uint64_t samplePos1616;
            uint32_t sampleInc1616;
            uint32_t sampleLoopStart;
            uint32_t sampleLoopEnd;
            uint32_t streamFracAccum;
        };
        // Mode: WAVE_CUSTOM (DSP filters, physical models)
        uint32_t cw[SYNTH_CUSTOM_WAVE_STATES]; 
    };
    // ...
};
```

---

## 6. Unified API Reference

### 1. Engine Initialization

```cpp
#include "ESP32Synth.h"

ESP32Synth synth;

void setup_audio() {
    // I2S Standard Output (PCM5102A on pins Data=2, BCK=4, WS=15 on ESP32)
    // S3 suggested pins: Data=5, BCK=4, WS=6
    synth.begin(2, SMODE_I2S, 4, 15, I2S_32BIT);

    // Single-pin PWM Output (10-bit resolution on GPIO 25)
    // synth.begin(25, SMODE_PWM, -1, -1, I2S_16BIT);

    // Single-pin PDM Output (Oversampled 16-bit stream on GPIO 2)
    // synth.begin(2, SMODE_PDM, 4, -1, I2S_16BIT);

    // Headless Output (Offline rendering / SD file bounce)
    // synth.beginHeadless(48000);

    synth.setMasterVolume(255); // 0-255 scale
}
```

### 2. Basic Voice & Pitch Control

Frequencies are defined in CentiHz ($1\text{ Hz} = 100\text{ cHz}$) to maintain integer precision. Note macros are provided via `ESP32SynthNotes.h`:

```cpp
// Play C4 (Middle C) on Voice 0 at maximum volume
synth.noteOn(0, c4, 255);

// Set wave shape and pulse width
synth.setWave(0, WAVE_PULSE);
synth.setPulseWidth(0, 128); // 50% duty cycle (0-255 scale)

// Master bitcrush reduction (0 = disabled, 1-32 = bit depth)
synth.setMasterBitcrush(8);

// Trigger release phase
synth.noteOff(0);
```

### 3. Modulations & Portamento

```cpp
// ADSR Envelope: Attack=10ms, Decay=150ms, Sustain=120, Release=1200ms
synth.setEnv(0, 10, 150, 120, 1200);

// Vibrato: 6.5 Hz rate, 30 Hz peak-to-peak depth
synth.setVibrato(0, 650, 3000);

// Tremolo: 4.0 Hz rate, depth scale 80 (0-255)
synth.setTremolo(0, 400, 80);

// Linear frequency slide to C5 over 500ms
synth.slideFreqTo(0, c5, 500);

// Arpeggiator: Voice 0, 120ms per step, [C4 -> E4 -> G4 -> C5]
synth.setArpeggio(0, 120, c4, e4, g4, c5);
```

---

## 7. The Power of `SMODE_PWM` (LEDC Bare-Metal Audio)

`SMODE_PWM` drives a hardware LEDC timer directly at the carrier frequency of **47,962 Hz**. 

The hardware interrupt (`ledc_ovf_isr`) runs entirely in IRAM, writing samples directly to the timer duty registers. This bypasses FreeRTOS task context switching completely, producing clean 10-bit audio on a single GPIO pin without external DAC hardware.

---

## 8. Dual-Framework Filesystem Streaming (SD Card)

WAV streaming uses a background task pinned to Core 0 that pre-fills a ring buffer, decoupling file I/O latency from audio generation:

```cpp
#ifdef ARDUINO
#include <SD.h>

void play_background_track() {
    // Voice 1, SD filesystem, filepath, volume, root pitch, loop enable
    synth.playStream(1, SD, "/ambient.wav", 255, c4, true);
}
#endif
```

Master output can also be dumped to a file in real-time:
```cpp
synth.startRecording(SD, "/recording.wav");
// ... audio processing ...
synth.stopRecording(); // Finalizes WAV header
```

---

## 9. External Protocol Pull Mode (A2DP Bluetooth & Wi-Fi)

For wireless audio streaming (such as Bluetooth A2DP, ESP-NOW, or WebSockets), initialize the engine in pull mode using `SMODE_CUSTOM`:

```cpp
ESP32Synth synth;

void setup() {
    synth.beginCustom(44100, nullptr); // Engine will not spawn an internal hardware task
    synth.noteOn(0, c4, 255);
}

// Callback invoked by your network or Bluetooth transmission stack
void write_bluetooth_packet(uint8_t *stream_buffer, int buffer_length) {
    int samplePairs = buffer_length / 4; // 16-bit interleaved stereo = 4 bytes per frame
    synth.generateSamplesStereo((int16_t*)stream_buffer, samplePairs);
}
```

---

## 10. Fixed-Point Advanced DSP & Custom Synthesis Blocks

ESP32Synth provides built-in algorithms in `ESP32Synth_Patches.hpp` covering anti-aliased oscillators, filters, instruments, and bus effects.

### Resonant Biquad Oscillators (PolyBLEP Filtered)

Version 2.4.7 provides two distinct biquad oscillator topologies:

#### 1. `DSP_BiquadOsc` / `DSP_BiquadOscHQ` (High-Precision TDF-II Q28)
Implements Transposed Direct Form II with 28-bit fixed-point coefficients and 64-bit internal accumulation. Includes sub-350 Hz polynomial approximation to prevent filter instability in lower frequency ranges.
* **CPU Load**: ~4.35% per voice.
* **Application**: Basslines, resonant sweeps, studio-quality leads.

```cpp
#include "ESP32Synth_Patches.hpp"

void play_hq_filtered_saw() {
    synth.setCustomWave(0, ESP32Patches::DSP_BiquadOsc);
    synth.setCustomParam(0, 0, 0);    // Wave Type: 0 = Saw, 1 = Pulse, 2 = Triangle
    synth.setCustomParam(0, 1, 440);  // Cutoff Frequency: 440 Hz (Range: 20 - 20000)
    synth.setCustomParam(0, 2, 150);  // Resonance Q: 1.50 (Q * 100)
    synth.setCustomParam(0, 3, 0);    // Mode: 0 = LPF, 1 = HPF, 2 = BPF, 3 = Notch
    
    synth.noteOn(0, c3, 255);
}
```

#### 2. `DSP_BiquadOscFast` (Direct Form I)
Calculated with 24-bit fixed-point math and 32-bit state registers.
* **CPU Load**: ~3.45% per voice.
* **Application**: High-polyphony accompaniment, chord stacks, resource-constrained setups.

---

### Modular Bus Filters & FX Chains

Auxiliary buses (1 to 3) support multi-slot effect processing before mixing down to the master output.

#### Bus Filter: `FX_BiquadFilter` (HQ) & `FX_BiquadFilterFast`
Process audio directly on an audio bus with selectable $Q$ or physical bandwidth modes:

| Parameter | Function | Description / Range |
| :--- | :--- | :--- |
| `ep[0]` | Cutoff / Center Freq | $20\text{ Hz}$ to $20000\text{ Hz}$ |
| `ep[1]` | Filter Mode | `0`: LPF, `1`: HPF, `2`: BPF, `3`: Notch |
| `ep[2]` | Resonance / Bandwidth | If `ep[3] == 0`: $Q \times 100$ (e.g., $70 = 0.707$). If `ep[3] == 1`: Bandwidth in Hz. |
| `ep[3]` | Calibration Unit Mode | `0`: Standard Q mode, `1`: Physical Bandwidth mode |
| `ep[4]` | Dry / Wet Mix | `0` or `255`: 100% Wet. `1` to `254`: Linear dry/wet blend |

```cpp
// Route Voice 0 to Bus 1
synth.setVoiceBus(0, 1);

// Add a 120Hz Bandwidth Notch Filter on Bus 1, Slot 0
synth.setBusFX(1, 0, FX_CUSTOM, ESP32Patches::FX_BiquadFilter);
synth.setBusFXParam(1, 0, 0, 1000); // Notch Center Frequency: 1000 Hz
synth.setBusFXParam(1, 0, 1, 3);    // Mode: Notch
synth.setBusFXParam(1, 0, 2, 120);  // Bandwidth: 120 Hz
synth.setBusFXParam(1, 0, 3, 1);    // Unit Mode: 1 (Bandwidth in Hz)
synth.setBusFXParam(1, 0, 4, 255);  // 100% Wet

// Mix Bus 1 to Master (Bus 0) at full volume
synth.setBusMix(1, 255);
```

#### Guitar and Organ Emulations
```cpp
// Karplus-Strong physical modeling guitar
synth.setCustomWave(1, ESP32Patches::EKS_Guitar);
synth.noteOn(1, e2, 255);

// Add Leslie Rotary Speaker effect to Master Bus
synth.setCustomDSP(ESP32Patches::DSP_HammondLeslie);
synth.setDSPParam(0, 2);   // Speed: 0 = Stop, 1 = Slow (Chorale), 2 = Fast (Tremolo)
synth.setDSPParam(1, 140); // Internal Spring Reverb Wet Mix (0 - 255)
```

---

## 11. The FLT Engine: Write Floats, Run Integers

The `fip` fixed-point class compiles floating-point DSP syntax into signed **Q8.24 32-bit hardware integer operations**.

### The 3 Rules of FLT:
1. **Range Bounds (-128.0 to 127.999)**: Never assign raw frequencies directly to `fip`. Work with normalized frequency ratios ($\frac{f}{\text{SampleRate}}$) fitting within $0.0$ to $0.5$.
2. **Literal Wrapping**: Always wrap constant floats: `fip(0.5f)`. The compiler evaluates these as integer literals at compile time with zero runtime cycles.
3. **Engine Boundaries**: Use `fip::fromPhase()`, `fip::fromParam()`, and `fip::fromEnv()` on entry. Call `.toAudio16()` when writing samples to the mix buffer.

```cpp
#include "ESP32Synth_FLT.hpp"

void myCustomOsc(Voice* vo, int32_t* mixBuffer, int samples, int32_t startEnv, int32_t envStep) {
    fip phase = fip::fromPhase(vo->phase);
    fip inc   = fip::fromPhase(vo->phaseInc);
    
    int32_t currentEnv = startEnv;
    int32_t volBase = ((uint32_t)vo->vol * vo->trmModGain) >> 8;

    for (int i = 0; i < samples; i++) {
        fip sample = fip::sin(phase * fip::two_pi());

        int32_t envSafe = currentEnv >> 14;
        envSafe &= ~(envSafe >> 31);
        int32_t finalVol = (envSafe * volBase) >> 14;

        mixBuffer[i] += (sample.toAudio16() * finalVol) >> 16;

        phase += inc;
        currentEnv += envStep;
    }
    vo->phase = (uint32_t)(phase.val << 8);
}
```

### Pre-Rendering Wavetables (`FLT_Baker`)
Bake mathematical formulas into RAM tables during `setup()` to save runtime CPU cycles:

```cpp
#include "ESP32Synth_FLT.hpp"

const void* tableData;

void setup() {
    // Generate 4096-point 16-bit wavetable in slot 0
    tableData = FLT_Baker::bakeWavetable(&synth, 0, [](fip phase) {
        fip fundamental = fip::sin(phase * fip::two_pi());
        fip overtone    = fip::sin(phase * fip::two_pi() * fip(3.0f)) * fip(0.3f);
        return fip::fast_tanh(fundamental + overtone);
    }, 4096, BITS_16);
}
```

---

## 12. Development Tools & Advanced Troubleshooting

### Python Conversion Tools (`/tools`)
* `WavetableMaker.py`: Converts mathematical equations or single-cycle audio files into aligned C header tables (`WAVE_WAVETABLE`).
* `WavToEsp32SynthConverter.py`: Converts transient audio files into 4-bit, 8-bit, or 16-bit aligned C arrays for sample playback.

### Hardware Troubleshooting
* **WDT Watchdog Triggers:** Ensure the ESP32 CPU frequency is explicitly set to **240 MHz** in your board configuration.
* **Filter Instability on Custom DSP:** If implementing custom feedback loops, clamp intermediate accumulators to $[-262143, 262143]$ to prevent 32-bit integer overflows during saturation.
* **LEDC Register Collision:** When running `SMODE_PWM`, ensure no peripheral libraries (such as servo or motor drivers) reconfigure LEDC Channel 0 or Timer 0 registers.

---

# Feel free to make and post videos, code, or suggestions; I'll be happy to see/read them!

<p align="center"><i>Believe in Jesus Christ❤️</i></p>
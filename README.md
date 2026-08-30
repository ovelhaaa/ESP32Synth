# ESP32Synth v2.4.6 — Highly Optimized Bare-Metal Synth Engine for Embedded Polyphony

<p align="center">
  <img src="https://raw.githubusercontent.com/danilogcrf2-oss/ESP32Synth/main/banner.jpg" alt="ESP32Synth banner" width="100%">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/version-2.4.6-green.svg" alt="Version">
  <img src="https://img.shields.io/badge/platform-ESP32%20%7C%20ESP32--S3%20%7C%20ESP32--S2%20%7C%20ESP32--C3%20%7C%20ESP32--C6-orange.svg" alt="Platform">
  <img src="https://img.shields.io/badge/framework-Arduino%20%7C%20ESP--IDF-blue.svg" alt="Framework">
  <img src="https://img.shields.io/badge/license-MIT-yellow.svg" alt="License">
</p>

A high-performance, polyphonic audio synthesis library for the ESP32 series (including S3, S2, C3, C6, etc.). Engineered for extreme bare-metal optimization, low-latency rendering, massive voice density, custom DSP hooks, and direct filesystem/SD-card streaming. Dual-framework support ensures compilation in both Arduino IDE and VS Code (PlatformIO) under either Arduino or native ESP-IDF.

### ✨ What's New in v2.4.6
* **The FLT Engine (Float Translator)**: Write DSP code using intuitive floating-point logic (`fip`) while the engine translates it into blazing-fast Q8.24 32-bit hardware integers at compile time! Zero CPU cost for float literals.
* **Wavetable Baker (`FLT_Baker`)**: Pre-calculate complex mathematical waveforms into RAM during setup using float math. Choose 16-bit, 8-bit, or 4-bit depths to save memory footprint.
* **Expanded DSP Patches**: New highly optimized instruments and FX added to `ESP32Synth_Patches.hpp`, including Karplus-Strong Guitar (`EKS_Guitar`), Hammond B3 Organ, FM Synthesis, Leslie Speaker FX, and a full Pedalboard FX chain.
* **Xtensa Core Micro-Optimizations**: Enhanced branch predictor hints (`LIKELY`/`UNLIKELY`) and deeper SIMD shielding, squeezing even more stability out of the Xtensa LX7 pipeline for maximum polyphony.

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

The extreme polyphony achievements of ESP32Synth (300+ voices on classic ESP32 chips, up to 500 on ESP32-S3) are not merely for playback metrics. This density serves as a **mathematical proof of efficiency**. 

By eliminating `float` operations, hardware divisions, and branch instructions from the hot audio rendering path, and exploiting architecture-specific hardware instructions like SIMD (`v4i32`) and hardware clamping (`CLAMPS`), we achieve extreme CPU headroom. This unused processing power allows developers to build highly complex synthesis blocks, such as:
* **6-Operator FM Synthesis** (emulating hardware like the Yamaha DX7)
* **Acoustic Physical Modeling** (Karplus-Strong string, waveguide, and drum-head modeling)
* **Adaptive Multi-Pole Resonant Filters (Biquads)**
* **PolyBLEP Anti-Aliased Waveforms**

To implement these blocks natively, you must maintain this performance philosophy: **use strictly 16.16, 24.8, or 32.32 fixed-point math, look-up tables (LUTs), and bitwise operations (`>>`).** *(Or, use the new FLT Engine to do it for you—see Section 11).*

---

## 2. PlatformIO (VS Code) & ESP-IDF Integration

With **v2.4.6**, PlatformIO integration is native. File system abstractions are unified, allowing you to run identical synth files under both Arduino and ESP-IDF frameworks.

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
Make sure to add ESP32Synth to your project's components or `src` directory.
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

While ESP32Synth is highly optimized for standard dual-core ESP32 chips operating at 240MHz, its hardware abstraction layers support the broader Espressif chip family.

### Core Allocation Architectures
* **Dual-Core SoC (Classic ESP32, ESP32-S3):** The high-priority DSP loop pins directly to Core 1 (`SYNTH_AUDIO_TASK_CORE 1`). This completely isolates the real-time audio thread from application execution, Bluetooth/Wi-Fi processing, or display routines on Core 0, enabling maximum polyphony.
* **Single-Core SoC (ESP32-S2, ESP32-C3, ESP32-C6, etc.):** The DSP task competes with other application threads. To maintain stable output, set the CPU frequency to its highest supported state. Limit polyphony parameters accordingly to avoid thread starvation.

### Hardware Output Mode (`SMODE`) Support Matrix

| Chip Model | SMODE_DAC | SMODE_I2S | SMODE_PDM | SMODE_PWM |
| :--- | :--- | :--- | :--- | :--- |
| **Classic ESP32** | Supported (GPIO 25, 26) | Supported | Supported | Supported (High-Speed LEDC) |
| **ESP32-S3** | Not Available | Supported | Supported (Ideal) | Supported (Low-Speed LEDC) |
| **ESP32-S2** | Supported (GPIO 17, 18) | Supported | Supported | Supported (Low-Speed LEDC) |
| **ESP32-C3 / C6** | Not Available | Supported | Supported | Supported (Low-Speed LEDC) |

*Note: `SMODE_HEADLESS` and `SMODE_CUSTOM` are software-driven and natively supported on all chip architectures.*

---

## 4. Core Configuration & Latency Tuning

Static parameters can be directly edited inside `ESP32Synth_Config.hpp` to customize RAM consumption and balance processing latency against overall polyphony stability.

### Resource Allocation Parameters
```cpp
// ESP32Synth_Config.hpp Core Limits
#define MAX_VOICES      80   // Maximum active concurrent synthesis voices.
#define MAX_WAVETABLES  20   // Maximum register space for custom wavetables.
#define MAX_SAMPLES     20   // Maximum registers for loaded RAM samples.
#define MAX_ARP_NOTES   16   // Maximum steps per individual voice arpeggiator.
#define MAX_STREAMS      4   // Maximum concurrent background SD file streams.
#define STREAM_BUF_SAMPLES 2048 // Streaming ring buffer length (must be a power of 2).
```

### Latency Optimization & DMA Tuning
You can calculate the processing latency using this formula:
$$\text{Latency (ms)} = \frac{\text{Buffer Length} \times \text{Buffer Count}}{\text{Sample Rate}} \times 1000$$

Configure these definitions directly inside `ESP32Synth_Config.hpp`:

* **High Polyphony / Robust Protection (Default):**
  * `SYNTH_DMA_BUF_LEN 512` | `SYNTH_DMA_BUF_COUNT 6` (Approx. 64ms latency).
* **Balanced / Real-Time MIDI:**
  * `SYNTH_DMA_BUF_LEN 256` | `SYNTH_DMA_BUF_COUNT 4` (Approx. 21ms latency).
* **Live Action / Ultra-Low Latency:**
  * `SYNTH_DMA_BUF_LEN 128` | `SYNTH_DMA_BUF_COUNT 2` (Approx. 5.3ms latency).

---

## 5. Memory Footprint & Hardware Isolation

### Voice Structure Optimization
To maximize RAM availability and prevent cache drops, ESP32Synth employs an extreme alignment strategy. Mutual exclusion is achieved via an explicit `union` block inside the `Voice` structure:

```cpp
struct Voice {
    int64_t slideVolCurr; // 8-byte alignment for fast Xtensa pipeline execution
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
        // Mode: WAVE_CUSTOM
        uint32_t cw[6]; // Exactly 24 bytes, balancing memory limits.
    };
    // ...
};
```
This union guarantees that regardless of your voice configuration, the core footprint of each voice does not exceed memory constraints, keeping the CPU Instruction Cache (ICache) hyper-optimized.

---

## 6. Unified API Reference

### 1. Engine Initialization

Choose the initialization method that corresponds to your hardware routing:

```cpp
#include "ESP32Synth.h"

ESP32Synth synth;

void setup_audio() {
    // Standard I2S Mode (External DAC like PCM5102A - BCK, WS, DATA)
    // Parameters: dataPin, mode, clkPin, wsPin, BitDepth
    synth.begin(5, SMODE_I2S, 4, 6, I2S_32BIT);

    // Or: Single-Pin Hardware PWM Mode (10-bit audio on pin 25)
    // synth.begin(25, SMODE_PWM, -1, -1, I2S_16BIT);

    // Or: PDM Mode (High-Frequency 1-bit oversampled audio on pin 2)
    // synth.begin(2, SMODE_PDM, 4, -1, I2S_16BIT);
    
    // Or: Headless Mode (No audio hardware, pure internal rendering for SD recording)
    // synth.beginHeadless(48000);

    // Set engine-wide volume (0-255 scaling)
    synth.setMasterVolume(255);
}
```

### 2. Basic Voice & Pitch Control

Pitch is controlled in hundredths of a Hz ("CentiHz") to achieve precise intonation using integers. (e.g., `c4`, `ds4`). Use the included `ESP32SynthNotes.h` macros.

```cpp
// Triggers voice 0 at C4 (Middle C), Volume 255
synth.noteOn(0, c4, 255);

// Update frequency and pulse-width dynamically
synth.setFrequency(0, cs4); // Shift pitch up to C#4
synth.setWave(0, WAVE_PULSE);
synth.setPulseWidth(0, 128); // 50% square duty cycle (0-255 scale)

// Set custom bitcrush resolution (0-32 bits, 0 means disabled)
synth.setMasterBitcrush(8); // Lo-fi 8-bit output reduction

// Triggers envelope release stage
synth.noteOff(0);
```

### 3. Modulations, Slides, and Arpeggios

We use Bresenham's algorithm for pitch slides to perform high-resolution portamento without hardware divisions.

```cpp
// Per-voice ADSR (Attack: 10ms, Decay: 150ms, Sustain Lvl: 120, Release: 1200ms)
synth.setEnv(0, 10, 150, 120, 1200);

// Vibrato (Frequency Modulation): LFO Rate 6.5Hz (650 cHz), LFO Depth 30Hz (3000 cHz)
synth.setVibrato(0, 650, 3000);

// Tremolo (Amplitude Modulation): LFO Rate 4Hz (400 cHz), LFO Depth 80
synth.setTremolo(0, 400, 80);

// Slide pitch to C5 over exactly 500 milliseconds
synth.slideFreqTo(0, c5, 500);

// Multi-step Arpeggiator (Voice 0, Step duration: 120ms, Notes: C4, E4, G4, C5)
synth.setArpeggio(0, 120, c4, e4, g4, c5);
```

---

## 7. The Power of `SMODE_PWM` (LEDC Bare-Metal Audio)

No external DAC? No problem. The PWM mode (`SMODE_PWM`) runs completely decoupled from traditional timers. We attach our interrupt handler (`ledc_ovf_isr`) directly to the LEDC timer's hardware overflow event.

Written in high-priority Assembly-level IRAM, the handler feeds duty-cycle updates straight to hardware registers, bypassing FreeRTOS scheduling overhead. This produces a clean carrier frequency locked to **47,962 Hz** with precise 10-bit resolution. Just add a simple RC low-pass filter to your pin!

---

## 8. Dual-Framework Filesystem Streaming (SD Card)

ESP32Synth natively translates filesystem calls based on the active compiler toolchain. The IO decoder runs on Core 0 inside a lower-priority background thread, loading a **Ring Buffer** to prevent SD card stalls from blocking the audio.

### Arduino Framework Stream (Uses `fs::FS`)
```cpp
#ifdef ARDUINO
#include <SD.h>

void play_background_track() {
    // Voice, FS Handle, Filepath, Volume, RootPitch, Loop
    synth.playStream(1, SD, "/ambient_music.wav", 255, c4, true);
}
#endif
```

### Real-Time SD Recording
You can record the master output bus to a `.wav` file on the SD card while it plays:
```cpp
// Starts an isolated DMA recording thread
synth.startRecording(SD, "/my_recording.wav"); 
// ... wait/play ...
synth.stopRecording(); // Safely closes and writes the WAV Header
```

---

## 9. External Protocol Pull Mode (A2DP Bluetooth & Wi-Fi)

To output audio over wireless connections (Bluetooth A2DP, ESP-NOW, or WebSockets), configure the engine in `SMODE_CUSTOM`. This turns off internal DMA timers and relies on a "Pull Mode" architecture.

```cpp
ESP32Synth synth;

void setup() {
    // Setup at 44.1kHz or 48kHz with no automatic timer (customOutput = nullptr)
    synth.beginCustom(44100, nullptr);
    synth.noteOn(0, c4, 255);
}

// Your wireless network or Bluetooth stack audio callback
void write_bluetooth_packet(uint8_t *stream_buffer, int buffer_length) {
    int samplePairs = buffer_length / 4; // Each 16-bit stereo frame is 4 bytes (L + R)
    // Under the hood, this converts, scales, and copies rendered frames directly
    synth.generateSamplesStereo((int16_t*)stream_buffer, samplePairs);
}
```

---

## 10. Fixed-Point Advanced DSP & Custom Synthesis Blocks

Inject complex physical effects and waveshapes into the engine. With **v2.4.6**, we include `ESP32Synth_Patches.hpp` offering professional anti-aliased oscillators, instruments, and complex FX chains!

### Using the Built-In Patches
Replaces standard raw oscillators with robust algorithms like the PolyBLEP anti-aliased 24-bit fixed-point resonant State Variable (RBJ) Biquad filter, Karplus-Strong Strings, or Hammond B3 simulations:

```cpp
#include "ESP32Synth_Patches.hpp"

void play_filtered_saw() {
    synth.setCustomWave(0, ESP32Patches::DSP_BiquadOsc);
    synth.setCustomParam(0, 0, 0);    // Wave: 0 = Saw, 1 = Pulse, 2 = Triangle
    synth.setCustomParam(0, 1, 800);  // Cutoff Freq: 800 Hz
    synth.setCustomParam(0, 2, 200);  // Resonance Q: 2.0 (Value * 100)
    synth.setCustomParam(0, 3, 0);    // Mode: 0 = LPF, 1 = HPF, 2 = BPF, 3 = Notch
    
    synth.noteOn(0, c4, 255);
}

void play_guitar() {
    synth.setCustomWave(1, ESP32Patches::EKS_Guitar); // Authentic Karplus-Strong string pluck
    synth.noteOn(1, e4, 255);
}
```

### Applying Master/Bus FX Chains
You can now apply effects like Leslie speakers or distortion pedalboards natively using the multi-bus architecture:
```cpp
// Route voice 0 to Bus 1
synth.setVoiceBus(0, 1);

// Apply a Pedalboard (Distortion + Chorus + Delay) to Bus 1, Slot 0
synth.setBusFX(1, 0, FX_CUSTOM, ESP32Patches::DSP_Pedalboard);
synth.setBusFXParam(1, 0, 0, 1); // Enable Distortion
synth.setBusFXParam(1, 0, 1, 1); // Enable Chorus
synth.setBusFXParam(1, 0, 2, 1); // Enable Delay

// Route Bus 1 to Master output (Volume 255)
synth.setBusMix(1, 255); 
```

### Building Your Own Custom DSP Voice
You can write completely new generation routines. Just respect the mathematical rule: No `float`, no hardware division in the loop. *(Or, use the new FLT Engine described below!)*

```cpp
// Extremely optimized String pluck (Legacy integer style Example)
void IRAM_ATTR myPluckOscillator(Voice* vo, int32_t* mixBuffer, int samples, int32_t startEnv, int32_t envStep) {
    int32_t currentEnv = startEnv;
    int32_t volBase = ((uint32_t)vo->vol * vo->trmModGain) >> 8;
    
    for (int i = 0; i < samples; i++) {
        // [ YOUR 16.16 FIXED-POINT MATH HERE ]
        int32_t signal = 0; // generate sample

        // Apply 32-bit Envelope & Volume Scale
        int32_t envSafe = currentEnv >> 14;
        envSafe &= ~(envSafe >> 31); // Absolute protection against negative clipping
        int32_t finalVol = (int32_t)((envSafe * volBase) >> 14);

        mixBuffer[i] += (signal * finalVol) >> 16;
        currentEnv += envStep;
    }
}
```

---

## 11. The FLT Engine: Write Floats, Run Integers (NEW in v2.4.6)

The `fip` (Fixed-Point) class acts as a transparent translator. It allows you to write custom DSP algorithms using familiar floating-point syntax (`fip::sin()`, `0.5f`), while the C++ compiler translates everything into **ultra-fast Q8.24 32-bit hardware integers** at compile time!

### The 3 Golden Rules of FLT:
1. **The Q8.24 Range Limit (-128.0 to 127.999)**: Never put raw frequency values (like `48000.0` or `440.0`) into a `fip`, as it will overflow. Always work with normalized ratios (Hz / SampleRate) which safely fit between 0.0 and 0.5. Example: `fip(440.0f / 48000.0f)`.
2. **Wrap Your Floats**: Never mix naked floats with `fip` in equations. Always wrap literal floats like `fip(0.5f)` so GCC optimizes the conversion to **zero** CPU cost at runtime.
3. **Gateway I/O**: Use `fip::fromPhase()`, `fip::fromParam()`, or `fip::fromEnv()` to bring engine variables into FLT math, and `.toAudio16()` to safely return the signal to the mix buffer.

### Example: FLT Custom Oscillator
```cpp
#include "ESP32Synth_FLT.hpp"

void myCustomOsc(Voice* vo, int32_t* mixBuffer, int samples, int32_t startEnv, int32_t envStep) {
    fip phase = fip::fromPhase(vo->phase);          // Gateway IN: Engine Phase -> 0.0 to 1.0
    fip inc   = fip::fromPhase(vo->phaseInc);             
    
    int32_t currentEnv = startEnv;
    int32_t volBase = ((uint32_t)vo->vol * vo->trmModGain) >> 8;

    for (int i = 0; i < samples; i++) {
        // Pure Q8.24 Integer Math disguised as Floats!
        fip myWave = fip::sin(phase * fip(2.0f) * fip::pi()); 
        
        int32_t envSafe = currentEnv >> 14;
        envSafe &= ~(envSafe >> 31);
        int32_t finalVol = (envSafe * volBase) >> 14;

        // Gateway OUT: fip -> Audio Int16
        mixBuffer[i] += (myWave.toAudio16() * finalVol) >> 16;  
        
        phase += inc;
        currentEnv += envStep;
    }
    vo->phase = (uint32_t)(phase.val << 8); // Save phase back to engine
}
```

### Baking Wavetables Mathematically (`FLT_Baker`)
If you don't need real-time parameter modulation, pre-calculate complex equations into RAM during `setup()` and play them directly as an ultra-fast wavetable!

```cpp
#include "ESP32Synth_FLT.hpp"

const void* bakedData;

void setup() {
    // ... begin synth ...

    // Runs ONCE during setup. Bakes the math into Wavetable Slot 0 (16-bit, 4096 points)
    bakedData = FLT_Baker::bakeWavetable(&synth, 0, [](fip phase) {
        fip wave1 = fip::sin(phase * fip::two_pi());
        fip wave2 = fip::cos(phase * fip::two_pi() * fip(3.0f)) * fip(0.5f);
        return fip::fast_tanh(wave1 + wave2); // Soft-clipper applied
    }, 4096, BITS_16);
}

void play_note() {
    synth.setWave(0, WAVE_WAVETABLE);
    synth.setWavetable(0, bakedData, 4096, BITS_16);
    synth.noteOn(0, c4, 255);
}
```

---

## 12. Development Tools & Advanced Troubleshooting

### Utility Scripts (`/tools`)
The repository contains two high-speed python utilities:
*   `WavetableMaker.py`: Converts complex sound mathematical equations or wave segments directly into static aligned C tables (`.h`) mapped as `WAVE_WAVETABLE`.
*   `WavToEsp32SynthConverter.py`: Converts short single-cycle audio files into 4-bit, 8-bit, or 16-bit aligned static memory arrays, avoiding the need for SD cards for transient instruments.

### Core-Level Debugging
* **WDT Reset / Starvation Jitter:** If you hear digital clicking or trigger Core Watchdog Resets, verify that the Xtensa processor is operating at **240MHz**. Standard ESP32 boards default to 160MHz in some configurations, significantly reducing available processing headroom.
* **FPU Contention on S3:** ESP32-S3 uses advanced vector SIMD registers on Core 1. If other intensive tasks (such as image analysis, cameras, or complex math) run concurrently on Core 1, task contention will occur. Configure standard tasks on Core 0 and preserve Core 1 exclusively for the synth engine.
* **Flickering PWM Audio:** Under `SMODE_PWM`, make sure no other task attempts to access LEDC Channel 0 or write to Timer 0 registers. This breaks the latch alignment of the overflow ISR. 

---

# Feel free to make and post videos, code, or suggestions; I'll be happy to see/read them!

<p align="center"><i>Believe in Jesus Christ❤️</i></p>

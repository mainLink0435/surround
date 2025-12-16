# 5.1 Surround Sound Integration for Libopenmpt

This project implements full **5.1 surround sound** (FL, FR, C, LFE, SL, SR) output for tracker module files (MOD, S3M, XM, IT, etc.) by modifying the underlying **libopenmpt** library and integrating the necessary API calls into the **audiodecoder.openmpt** add-on wrapper.

## Project Goal

The primary objective is to enable 6-channel playback on hardware such as the Vontar X4 (Aarch64), ensuring that tracker modules utilize the full surround field while preserving the original stereo mixing behavior when needed.

## Architecture & Constraints

### Core Principles

* **Preserve Stereo:** All existing stereo mixing functionality is preserved. New surround code is implemented by extending or wrapping existing functions. Public functions handling stereo output remain unmodified.
* **Library Integration:** The custom, surround-enabled `libopenmpt` is statically linked directly into the `audiodecoder.openmpt` add-on binary to prevent clashes with system libraries.
* **API Extension:** A new public function/API path is exposed to explicitly request 6-channel output.
* **Configuration:** The user-facing 'Stereo' vs. 'Surround' configuration switch is handled entirely within the add-on.

### Build Environment

* **Development:** Linux x64 (compilation checks).
* **Target:** Aarch64 (Vontar X4) for actual deployment and audio testing.

## Technical Implementation Features

### 1. Core 6-Channel Mixer

**Status:** ✅ Implemented & Tested

The mixer architecture has been refactored to support a 6-channel pipeline alongside the existing stereo pipeline.

* **Independent Mixing:** The `MixChannelSurround()` function handles mixing without reusing stereo code, preventing "hacky" conversions.
* **Buffer Structure:** Uses 6 non-interleaved buffers (FL, FR, C, LFE, SL, SR) instead of a single interleaved buffer during the mix process.
* **Direct Accumulation:** Samples accumulate directly to surround buffers.
* **Stability Fixes (Dec 5):**
  * **Buffer Offset Desync:** Fixed robotic artifacts by ensuring `surrBufOffset` advances even when skipping silent samples.
  * **Static Noise:** Implemented strict zeroing of the `tempStereo` buffer before every mix call to prevent stack garbage accumulation.
  * **Pointer Logic:** Corrected buffer pointer reset logic inside mix loops.

**Key Files Modified:**

* `openmpt/soundlib/Sndfile.h`: Added 6 surround buffers, `CreateSurroundMix` declaration.
* `openmpt/soundlib/Fastmix.cpp`: Implemented `MixChannelSurround`.
* `openmpt/soundlib/Sndmix.cpp`: Added 6-channel interleaving logic.

### 2. Low-Frequency Effects (LFE) Channel

**Status:** ✅ Implemented & Tested

The LFE channel is generated post-mix using a crossover filter on the summed signal.

* **Source:** Sum of all 5 main channels (FL + FR + C + SL + SR).
* **Filter:** 2nd-order Butterworth IIR lowpass filter.
* **Cutoff:** 120Hz (Q = 0.7071).
* **Attenuation:** -10dB (Linear gain = 0.3162) applied to prevent clipping on summed bass content.
* **Implementation:** Direct Form II Transposed for numerical stability.

### 3. Panning Logic & Modes

**Status:** ✅ Linear & FT2 Implemented | 🔄 Soft Panning Pending

The system maps standard tracker pan values (0-255) to the 5-speaker array (SL, FL, C, FR, SR).

**Pan Position Mapping:**

* `0`: Surround Left (SL)
* `64`: Front Left (FL)
* `128`: Centre (C)
* `192`: Front Right (FR)
* `255`: Surround Right (SR)

**Supported Modes:**

1. **Linear Panning:** Simple linear crossfade between adjacent speakers.
2. **FT2 (Constant-Power):** Uses lookup tables (derived from `XMPanningTable` but extended to 5 channels) to maintain constant perceived power. Static lookup tables were generated via Python script and integrated into `Tables.cpp`.
3. **Soft Panning:** (Planned) Keeps one speaker at 50% minimum during crossfades to reduce perceived volume dips.

**Implementation Notes (Dec 6):**

* **Gain Correction:** Resolved early issues with double volume application in `MixSurroundNoRamp`. Gains are now applied directly without extra scaling.
* **Ramping:** Fixed normalization bugs in `MixSurroundRamp` to ensure volume levels match stereo output.

### 4. Divergence / Bleed Matrix

**Status:** ✅ Implemented

To solve the "isolated speaker" problem where sound only comes from one specific driver, a **Horseshoe Divergence Matrix** is implemented. This blends energy between neighboring speakers.

**Topology:**

* **Constant:**`kDiv = 0.25f` (-12dB bleed).
* **Logic:**
  * `SL` receives bleed from `FL`.
  * `FL` receives bleed from `SL` and `C`.
  * `C` receives bleed from `FL` and `FR`.
  * `FR` receives bleed from `SR` and `C`.
  * `SR` receives bleed from `FR`.
* **Constraint:** SL and SR do not bleed into each other, nor do they bleed into C.

**Mathematical Model:**

```
int32 finalSL = baseSL + (baseFL * kDiv);
int32 finalFL = baseFL + (baseSL * kDiv) + (baseC * kDiv);
int32 finalC  = baseC  + (baseFL * kDiv) + (baseFR * kDiv);
int32 finalFR = baseFR + (baseSR * kDiv) + (baseC * kDiv);
int32 finalSR = baseSR + (baseFR * kDiv);
// Values are clamped to 65536
```

## Development & Contribution

### Verification Checklist (TDD)

Before declaring a phase complete, the following must be verified:

1. **Variable Initialization:** Trace every new variable from declaration to usage.
2. **Data Flow:** Trace execution from API entry (`read_interleaved_5point1`) to final output.
3. **Runtime Check:** Verify calculations with realistic inputs (mental walk-through).
4. **Defensive Programming:** Investigate all conditional checks and flags.

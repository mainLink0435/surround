PROJECT BRIEF: 5.1 SURROUND FOR LIBOPENMPT INTEGRATION
======================================================

🎯 Project Goal
---------------

To implement full **5.1 surround sound** (FL, FR, C, LFE, SL, SR) output for tracker module files (MOD, S3M, XM, IT, etc.) by modifying the underlying **libopenmpt** library and integrating the necessary API calls into the **audiodecoder.openmpt** add-on wrapper.

⚙️ Architectural & Non-Negotiable Constraints
-----------------------------------------------

* **Preserve Stereo:** All existing stereo mixing functionality must be **preserved and fully operational**. New surround code must be implemented by **duplicating, extending, or wrapping** existing functions/classes. **Do not modify any existing public functions** that handle stereo output.
* **Library Integration:** The custom, surround-enabled libopenmpt **must be statically linked directly into the audiodecoder.openmpt add-on binary**. This is mandatory to prevent clashes with the system's existing library and ensures the custom code is always used.
* **API Extension:** libopenmpt must expose a new public function or API path that the add-on can call to explicitly request **6-channel output**.
* **Config Location:** The user-facing 'Stereo' vs. 'Surround' configuration switch will be handled **entirely within the audiodecoder.openmpt add-on**. The add-on will determine which libopenmpt function (stereo or new surround) to call.

💻 Phase 1: libopenmpt Implementation Details
---------------------------------------------

The following core features must be introduced into the libopenmpt source code:

### 1\. API and Core Structure

* Duplicate the internal mixing logic (classes/functions) responsible for 2-channel stereo output and adapt it to handle **6 output channels (5.1)**.
* Add a new function signature to the public C API (and/or C++ interface) that accepts parameters to request 6-channel output in the standard order: FL, FR, C, LFE, SL, SR.

### 2\. Surround Panning Law (0-255 Map)

Implement panning logic that maps the tracker pan value (0-255) to five main speakers (SL, FL, C, FR, SR) with crossfades between adjacent channels.

**Pan Position Mapping:**
* Pan Value **0**: **100% Surround Left (SL)**
* Pan Value **64**: **100% Front Left (FL)**
* Pan Value **128**: **100% Centre (C)**
* Pan Value **192**: **100% Front Right (FR)**
* Pan Value **255**: **100% Surround Right (SR)**
* Intermediate values: crossfade between adjacent speakers

**Panning Modes (matching stereo implementation):**

1. **Linear Panning (NoSoftPanning/default):**
   - Simple linear crossfade between adjacent speakers
   - Pan ranges: 0-64 (SL→FL), 64-128 (FL→C), 128-192 (C→FR), 192-255 (FR→SR)
   - Formula per segment: `speakerA_gain = (segmentMax - pan) / segmentRange`, `speakerB_gain = (pan - segmentMin) / segmentRange`
   - Example: pan=32 in SL→FL segment: SL gets 50%, FL gets 50%

2. **Soft Panning:**
   - Keeps one speaker at 50% minimum when crossfading
   - Reduces perceived volume loss at extreme pan positions
   - Per segment: when fading from A to B, A stays at min 50% until center, then B rises to 50%
   - Similar to stereo soft panning but extended across 5 speakers

3. **FT2 Panning (Square-Root/Constant-Power):**
   - Uses lookup table for square-root panning curve
   - `SurroundPanningTable` (5 speakers × 256 positions) is implemented and used for FT2 panning mode
   - Based on `XMPanningTable` used in stereo, but extended to 5-channel surround field
   - Provides most natural-sounding panning with constant perceived power

**Implementation Notes:**
- All three modes follow same structure as stereo panning in `Sndmix.cpp` lines 2534-2563
- Check `m_PlayConfig.getPanningMode()` to determine which algorithm to use
- Pan value comes from `chn.nRealPan` (already calculated, range 0-256)
- Apply panning AFTER resampling, when distributing tempStereo to surround buffers

### 3\. Low-Frequency Effects (LFE) Channel

* Implement a low-pass filter (e.g., a simple first or second-order IIR filter with a standard cutoff of 120Hz).
* In the new surround mixing function, the LFE channel (.1) must be populated by:
  1. Calculating the sum of the five main channels (FL, FR, C, SL, SR) after they have been panned.
  2. Applying the implemented low-pass filter to this summed signal.
  3. Applying a mandatory volume attenuation using a **linear gain factor of 0.3162** (equivalent to -10dB) to the filtered signal before outputting it to the LFE channel.

**Note:** LFE implementation will be done after panning logic is complete and tested.

### 4.  Understanding the architecture

* This dev machine is Linux x64. It's ok to build on this machine to check compilation
* The actual machine running the addon is a Vontar X4, and therefore aarch64. These modules will be built on that machine manually.

### The approach

* we are going to take an incremental TDD approach to this. implement one step, then i will build and test on the aarch64 device.
* we need to follow the existing software architecture and replicate the structure as much as possible.

### keep track

* intiially, create a "completed_steps.md" file, which outlines the work already done in the audiodecoder.openmpt project, and the openmpt project. the audiodecoder work is pretty final, there's not a lot that needs to be done there. there's plenty in the libopenmpt module, though. initially, follow the code through from readpcm in audiodecoder.openmpt where msurround = true, to work out where things are currently.
* then, as we iterate through steps, provide a git commit text, and also update the completed_steps.md file.
* do not utilise existing code for the mixing. utilising any of the stereo or quad mixing code is wrong.
* Never, ever run make clean. always just manually delete the problem file.


### current surround behavior snapshot (Dec 6, 2025)

- Independent surround mixer (`MixChannelSurround`) with linear and FT2/constant-power 5-point pan (SL→FL→C→FR→SR) and LFE lowpass (-10dB at 120Hz).
- Ramped surround mixers recompute pan gains per-sample using ramped L/R volumes to smooth pan changes.
- Short attack fade (~1 ms) when starting from silence to mask sample-edge clicks at low levels.
- FT2/constant-power panning is now active and selectable; soft panning remains pending.

### CRITICAL: Code Review and Verification Requirements

**MANDATORY before declaring any phase complete:**

1. **Variable Initialization Verification:**

   - Every new variable MUST be traced from declaration → initialization → usage
   - Search for initialization code in constructors, reset functions, or initialization blocks
   - Never assume variables are initialized - explicitly verify in the code
2. **Complete Data Flow Tracing:**

   - Trace the ENTIRE execution path from API entry point through all intermediate functions to final output
   - Verify data flows through all expected code paths with actual values
   - Test conditional branches - what happens if the condition is true? false? with uninitialized data?
3. **Runtime Execution Verification:**

   - Mentally execute the code with realistic input values
   - Verify each calculation produces expected ranges and values
   - Check for undefined behavior (uninitialized variables, division by zero, buffer overruns)
4. **Suspicious Pattern Investigation:**

   - Question every conditional check (e.g., "why check >= 0?")
   - Investigate defensive programming patterns - are they fully implemented?
   - If something looks like a flag, verify the flag semantics are properly implemented
5. **End-to-End Path Verification:**

   - Starting from API entry (e.g., `read_interleaved_5point1()`)
   - Trace through EVERY function call, variable assignment, and conditional
   - Verify the complete path from input → processing → output works with actual data

**If ANY of these checks cannot be verified, the code is NOT complete.**

## Git Workflow: Avoiding Full Rebuilds

### Problem

When reverting commits with `git reset --hard HEAD~N`, CMake detects that source files have changed timestamps and forces a full rebuild of everything. This is time-consuming and unnecessary when only testing previous states.

### Solution: Use `git commit --amend` Instead

**For iterative development (recommended):**

```bash
# Make your changes to files
git add .
git commit --amend --no-edit
git push origin master --force-with-lease
```

This updates the current commit in-place rather than creating new commits or reverting. The build system treats it as continuous work on the same commit, avoiding forced rebuilds.

### If You Must Go Back in History

If you absolutely need to revert to a previous commit to compare or test, use this approach to minimize rebuild time:

```bash
# Fetch the commit you want to revert to (replace COMMIT_HASH_TO_REVERT with actual hash)
git revert COMMIT_HASH_TO_REVERT

# Or, if you need to reset locally without pushing:
git reset --hard PREVIOUS_COMMIT_HASH

# After reset, update timestamps on all object files so make doesn't rebuild
find build -name "*.o" -exec touch {} \;

# Now rebuild - only modified source files will recompile
make
```

**Important:** The `find build -name "*.o" -exec touch {} \;` command updates the object file timestamps, tricking the build system into thinking they're still current. This works even if you went back in git history, because you're manually updating the .o file timestamps to match the "current" time.

### Why This Happens

- CMake tracks file modification timestamps to determine what needs rebuilding
- When git changes files (via reset/checkout), their timestamps change to the current time
- The build system sees "newer" source files and rebuilds everything that depends on them
- Using `git commit --amend` avoids this because you're modifying files in-place, keeping a consistent timeline

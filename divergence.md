Here is a precise, technically detailed prompt you can feed into your AI code assistant (Cursor, Copilot, etc.) to get the exact refactoring we discussed.

-----

### Prompt for AI Code Assistant

**Context:**
I am modifying the `ReadNote()` function in a C++ audio mixing engine (based on libopenmpt). The current code calculates 5.1 surround panning gains for tracker modules (MOD, S3M) and assigns them directly to channel variables (`chn.newSurroundSL`, etc.) inside conditional blocks.

**The Goal:**
I need to refactor this section to implement a **"Divergence/Bleed Matrix"** to solve the "isolated speaker" problem. The signal flow must change from `Calculation -> Assignment` to `Calculation -> Matrix Mixing -> Assignment`.

**Task Instructions:**

1.  **Refactor Variable Storage:**

      * Before the `if(panningMode == ...)` block, declare five temporary integer variables: `baseSL`, `baseFL`, `baseC`, `baseFR`, `baseSR`, and initialize them to `0`.
      * Update all three existing panning branches (Soft Panning, FT2 Panning, and the Linear `else` block) to assign their calculated values to these `base` variables instead of writing directly to `chn.newSurroundXX`.

2.  **Implement the Horseshoe Divergence Matrix:**

      * Immediately *after* the `if/else` block closes, implement a divergence matrix that blends energy between neighboring speakers.
      * Define a constant `float kDiv = 0.25f;` (representing -12dB bleed).
      * Calculate `final` values using the following **Horseshoe Topology** (Uni-directional end-caps):
          * **SL** receives bleed from **FL**.
          * **FL** receives bleed from **SL** and **C**.
          * **C** receives bleed from **FL** and **FR**.
          * **FR** receives bleed from **SR** and **C**.
          * **SR** receives bleed from **FR**.
      * *Constraint:* SL and SR must **not** bleed into each other, and they must **not** bleed into C.

3.  **Final Assignment & Clamping:**

      * Assign the `final` values to the actual channel variables (`chn.newSurroundSL`, etc.).
      * Apply a safety clamp using `std::min` to ensure the values do not exceed `65536` (as the bleed addition might push 100% signals over the limit).

**Mathematical Logic for the Matrix:**

```cpp
// Use this logic for the matrix step:
int32 finalSL = baseSL + (baseFL * kDiv);
int32 finalFL = baseFL + (baseSL * kDiv) + (baseC * kDiv);
int32 finalC  = baseC  + (baseFL * kDiv) + (baseFR * kDiv);
int32 finalFR = baseFR + (baseSR * kDiv) + (baseC * kDiv);
int32 finalSR = baseSR + (baseFR * kDiv);
```

**Generate the full refactored code block replacing the provided snippet.**
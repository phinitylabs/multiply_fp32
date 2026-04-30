# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

This design currently targets:
- **Bit-accurate results for normal FP32 numbers** (typical IEEE-754 behavior with round-to-nearest-even),
- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
- The design behaves as: z = a*b 
- z, a and b are single precision 32-bit IEEE-754 numbers

---

## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`   | in | 1 | Clock |
| `rst`   | in | 1 | Async reset (posedge) |
| `valid` | in | 1 | **1-cycle start pulse**; accepted only when not busy |
| `a`     |  in | 32 | Operand A (FP32 bits) |
| `b`     | in | 32 | Operand B (FP32 bits) |
| `z`         | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | **1-cycle pulse** when `z` is updated/valid |

### Handshake contract
- When `busy==0`, a high `valid` on a rising edge **starts** an operation:
  - `a` and `b` are **registered** into internal regs `a_r` and `b_r`.
  - The FSM begins at `counter = 1`.
- While `busy==1`, new `valid` pulses are **ignored**.
- When the operation completes:
  - `z` is updated,
  - `out_valid` pulses high for 1 clock cycle,
  - `busy` is cleared.

---

## Latency and Throughput

### Latency
- Fixed latency of **7 stages**.
- In this implementation the operation begins at stage `counter=1` and completes at `counter=7`.
- `out_valid` asserts on the cycle where stage 7 packing finishes.

A safe expectation for system-level timing is:
- **`out_valid` occurs 7 clock cycles after the start edge** (the clock edge where `valid` was sampled when idle).

### Throughput
- **Not pipelined** (single-issue).
- Max throughput is **1 result per 7 cycles** (assuming `valid` is asserted only when idle).

---

## Internal Data Model (IEEE-754 binary32)
For each operand:
- `sign` = bit 31
- `exp`  = bits 30:23 (biased exponent)
- `mant` = bits 22:0 (fraction)

Internal signals:
- `a_s, b_s, z_s`: sign bits
- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, used with `$signed`)
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
- `product`: 50-bit product of mantissas
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE

---

## Implementation Clarifications (Non-invasive Addendum)

This section does NOT modify the specification.  
It only clarifies intended interpretation to avoid ambiguity during implementation.

---

### Input Assumptions
- Inputs are restricted to NORMAL FP32 values:
  - exponent ∈ [1..254]
  - no zero, subnormal, inf, or NaN
- Therefore hidden bit is always 1

---

### Mantissa Handling
- Mantissas should be treated as 24-bit values before multiplication:

  a_m = {1'b1, a_r[22:0]}  
  b_m = {1'b1, b_r[22:0]}  

---

### Multiply Stage Clarification
Although Stage 4 mentions exponent adjustment and scaling:

- Implement multiplication as:
  
  z_e = a_e + b_e  
  product = a_m * b_m  

- Avoid applying both `+1` and `*4` together, as this leads to incorrect scaling

---

### Normalization Guidance
- Use MSB of product to determine normalization:

  if (product[47] == 1) → shift right and increment exponent  
  else → no shift  

---

### Bit Extraction Guidance
- Extract mantissa and rounding bits relative to MSB position  
- Avoid fixed slicing without checking normalization condition  

---

### Rounding (RNE)
- Apply round-to-nearest-even:

  if (guard && (round || sticky || LSB))  
      increment mantissa  

---

### Stage 3 Note
- For normal inputs, Stage 3 typically has no effect and may be treated as pass-through

---

### Underflow Simplification
- If final exponent ≤ 0 → output zero  
- Gradual underflow handling is not required for this task  

---

### Priority Note
If any ambiguity arises between stage descriptions and implementation:

- Follow a consistent interpretation across all stages  
- Avoid mixing multiple scaling/normalization approaches simultaneously  

---

## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.

### Stage 1 — Unpack
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form: `exp - 127`.
- Capture signs.

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- For normal operation:
  - **CRITICAL:** Use 24-bit registers for a_m and b_m.
- You must explicitly prepend the hidden bit: `a_m = {1'b1, a_frac}` and `b_m = {1'b1, b_frac}`.
- This 24-bit significand is what must be passed to the Stage 4 multiplier.

> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.
   
### Stage 3 — Input normalization (lightweight)
- If mantissa MSB is not set, shift left and decrement exponent.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.
     
### Stage 4 — Multiply core
- Compute result sign: `z_s = a_s ^ b_s`
- Exponent add: `z_e = a_e + b_e + 1`
- Mantissa product: `product = a_m * b_m * 4`
- **CRITICAL:** The intermediate `product` register must be **at least 50 bits wide**. Multiplying two 24-bit numbers creates a 48-bit result, and the `* 4` scaling requires 2 extra bits.
  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

### Stage 5 — Extract mantissa + rounding bits
- `z_m = product[49:26]`
- `guard_bit = product[25]`
- `round_bit = product[24]`
- `sticky = OR(product[23:0])`

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)
This stage performs:
1. **Underflow alignment** toward exponent -126:
   - Computes shift amount `sh = (-126 - z_e)` when `z_e < -126`.
   - Shifts mantissa right and accumulates shifted-out bits into sticky.
2. **Normalize** if MSB missing:
   - Left-shifts mantissa while adjusting exponent, carrying guard into LSB.
3. **RNE rounding**:
   - If `G == 1` and `(R || S || LSB)` then increment mantissa.
   - Handles carry-out from rounding:
     - If rounding overflows mantissa, set mantissa to 0x800000 and increment exponent.

### Stage 7 — Pack
- For normal path:
  - Pack sign, biased exponent, fraction.
  - If exponent indicates overflow -> output INF.
  - If exponent indicates exact denorm boundary -> force exponent field to 0 (denormal/zero representation).
- Asserts `out_valid` for one cycle and clears `busy`.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands
### Additional Clarifications for Correctness

#### Normalization and Rounding Order
- Mantissa normalization MUST be completed before rounding
- Rounding must be applied only after the mantissa is in normalized form
- Guard, round, and sticky bits must be derived from the normalized product
- Do not perform rounding on unnormalized mantissa values

#### Single Normalization Rule
- Perform normalization exactly once before rounding
- After rounding, only adjust if mantissa overflow occurs (shift right + increment exponent)
- Do not perform multiple normalization passes

#### Underflow Handling
- If the final exponent (after normalization and rounding) is less than or equal to 0:
  - The output must be exactly zero
  - Mantissa must be 0
  - Only sign bit may remain

#### Overflow Handling
- If the final exponent exceeds maximum representable value:
  - Output must be infinity (exp = 255, mantissa = 0)
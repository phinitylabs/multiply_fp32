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

## Required Operation Ordering  - jas

The implementation must follow this exact ordering for all normal inputs:

1. Unpack operands
2. Convert exponent to unbiased domain
3. Insert hidden leading 1 for normalized operands
4. Multiply mantissas
5. Compute result exponent
6. Extract mantissa and rounding bits
7. Normalize mantissa if needed
8. Apply round-to-nearest-even (RNE)
9. Handle rounding carry overflow
10. Handle overflow/underflow conditions
11. Pack IEEE-754 result

Do not reorder normalization, rounding, or exponent adjustment.
Normalization must happen before rounding.
Rounding carry propagation must happen before final exponent packing.

---

## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.

### Stage 1 — Unpack
- Register inputs `a` and `b` into `a_r` and `b_r`.
- Extract:
  - sign = bit 31
  - exponent = bits [30:23]
  - mantissa = bits [22:0]
- Convert exponent to unbiased representation:

  `unbiased_exp = exponent - 127`

- Mantissa is initially stored as:

  {1'b0, fraction}

Hidden-one insertion occurs later.

### Stage 2 — Classification + Hidden-One Setup
- For normal numbers:
  - exponent field is always within [1..254]
  - hidden leading one must always be inserted:

    mantissa[23] = 1

- For this task, assume only normal inputs are generated.
- NaN, INF, zero, and subnormal handling may still exist in RTL but are not required for correctness.

### Stage 3 — Input normalization (lightweight)
- If mantissa MSB is not set, shift left and decrement exponent.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply Core
- Result sign:

  `z_s = a_s ^ b_s`

- Result exponent:

  `z_e = a_e + b_e`

- Mantissa multiply:

  `product = a_m * b_m`

- `a_m` and `b_m` are 24-bit mantissas including hidden-one.
- `product` is a 48-bit multiplication result.

Important:
- Do not pre-normalize product.
- Do not shift product during multiplication.
- Normalization occurs later.

### Stage 5 — Mantissa Extraction
- Product is interpreted as a 48-bit value.

For normalized operands:
- product[47] determines whether normalization shift is required.

Extraction rule:

If product[47] == 1:
- `mantissa_candidate = product[47:24]`
- guard = product[23]
- round = product[22]
- sticky = OR(product[21:0])
- increment exponent by 1

Else:
- `mantissa_candidate = product[46:23]`
- guard = product[22]
- round = product[21]
- sticky = OR(product[20:0])

This extraction occurs before rounding.

### Stage 6 — Normalize + Round-to-Nearest-Even

Normalization:
- After extraction, mantissa must contain a leading 1 in bit[23].
- If leading 1 is missing:
  - left shift mantissa
  - decrement exponent

Round-to-Nearest-Even (RNE):

Definitions:
- G = guard bit
- R = round bit
- S = sticky bit
- LSB = mantissa[0]

Rounding rule:

Increment mantissa if:

G == 1 AND (R == 1 OR S == 1 OR LSB == 1)

This implements IEEE-754 ties-to-even behavior.

After increment:
- If mantissa overflows beyond 24 bits:
  - right shift mantissa by 1
  - increment exponent

Rounding must happen only after normalization.

### Stage 7 — Pack Result
- Convert unbiased exponent back to biased form:

  `exponent_out = z_e + 127`

Overflow:
- If `exponent_out >= 255`:
  - output infinity

Underflow:
- If `exponent_out <= 0`:
  - output zero

Final output:

z = {
  sign,
  `exponent_out[7:0]`,
  mantissa[22:0]
}

- Assert `out_valid` for exactly one cycle.
- Clear busy.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands,

---

## Simplified Assumptions for This Task

The hidden verification primarily targets normal FP32 multiplication.

Assume:
- Inputs are normal IEEE-754 numbers
- No NaN inputs
- No infinity inputs
- No denormal inputs
- No signed zero corner cases

Correctness is determined by:
- Proper exponent computation
- Correct mantissa normalization
- Correct RNE rounding
- Proper IEEE-754 packing

---

diff --git a/doc/spec.md b/doc/spec.md
index 47afeeffcc96b9e26d897a9699113a782aad69de..171185b1d7253066b916bc281c5772a63a20b49e 100644
--- a/doc/spec.md
+++ b/doc/spec.md
@@ -1,133 +1,115 @@
-# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)
+# fmultiplier — FP32 Multiplier (AI-Friendly Task Spec)
 
-## Overview
-`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.
+## 1) Objective
+Implement module `fmultiplier` that computes IEEE-754 single-precision multiplication:
 
-This design currently targets:
-- **Bit-accurate results for normal FP32 numbers** (typical IEEE-754 behavior with round-to-nearest-even),
-- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
-- The design behaves as: z = a*b 
-- z, a and b are single precision 32-bit IEEE-754 numbers
+- Mathematical behavior: `z = a * b`
+- Data format: `a`, `b`, and `z` are 32-bit IEEE-754 binary32 bit patterns
+- Interface uses a simple request/response handshake (`valid` in, `out_valid` out)
 
----
-
-## Interface
-
-### Ports
-| Port | Dir | Width | Description |
-|------|-----|-------|-------------|
-| `clk`   | in | 1 | Clock |
-| `rst`   | in | 1 | Async reset (posedge) |
-| `valid` | in | 1 | **1-cycle start pulse**; accepted only when not busy |
-| `a`     |  in | 32 | Operand A (FP32 bits) |
-| `b`     | in | 32 | Operand B (FP32 bits) |
-| `z`         | out | 32 | Result (FP32 bits) |
-| `out_valid` | out | 1 | **1-cycle pulse** when `z` is updated/valid |
-
-### Handshake contract
-- When `busy==0`, a high `valid` on a rising edge **starts** an operation:
-  - `a` and `b` are **registered** into internal regs `a_r` and `b_r`.
-  - The FSM begins at `counter = 1`.
-- While `busy==1`, new `valid` pulses are **ignored**.
-- When the operation completes:
-  - `z` is updated,
-  - `out_valid` pulses high for 1 clock cycle,
-  - `busy` is cleared.
+This task is graded by black-box tests. Matching observable behavior is more important than matching any specific internal micro-architecture.
 
 ---
 
-## Latency and Throughput
-
-### Latency
-- Fixed latency of **7 stages**.
-- In this implementation the operation begins at stage `counter=1` and completes at `counter=7`.
-- `out_valid` asserts on the cycle where stage 7 packing finishes.
-
-A safe expectation for system-level timing is:
-- **`out_valid` occurs 7 clock cycles after the start edge** (the clock edge where `valid` was sampled when idle).
-
-### Throughput
-- **Not pipelined** (single-issue).
-- Max throughput is **1 result per 7 cycles** (assuming `valid` is asserted only when idle).
+## 2) Required module interface
+
+```systemverilog
+module fmultiplier(
+    input  wire        clk,
+    input  wire        rst,
+    input  wire        valid,
+    input  wire [31:0] a,
+    input  wire [31:0] b,
+    output reg  [31:0] z,
+    output reg         out_valid
+);
+```
+
+### Reset behavior
+- `rst` is active-high.
+- On reset: clear internal state, set `out_valid=0`. `z` may be cleared to 0.
+
+### Handshake behavior
+- Accept a request only when not busy.
+- Latch `a` and `b` on the accepting cycle.
+- Ignore new `valid` pulses while busy.
+- Assert `out_valid` for exactly **one cycle** when result `z` is ready.
+
+### Latency requirement
+- Use **fixed latency of 7 cycles** from accepted request to `out_valid` pulse.
+- Throughput is single-issue (no overlap required).
 
 ---
 
-## Internal Data Model (IEEE-754 binary32)
-For each operand:
-- `sign` = bit 31
-- `exp`  = bits 30:23 (biased exponent)
-- `mant` = bits 22:0 (fraction)
+## 3) Functional correctness requirements
 
-Internal signals:
-- `a_s, b_s, z_s`: sign bits
-- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, used with `$signed`)
-- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
-- `product`: 50-bit product of mantissas
-- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE
+Your result must match IEEE-754 binary32 multiply semantics for tested cases.
 
----
+### Must-handle classes
+1. **Normal × Normal**
+2. **Zero** inputs (including signed zero)
+3. **Infinity** inputs
+4. **NaN** inputs
+5. Subnormal boundary behavior (underflow region) should be handled sensibly
 
-## FSM / Pipeline Stages
+### Special-case rules (minimum expected)
+- If either input is NaN -> return quiet NaN (`exp=255`, mantissa nonzero).
+- `Inf * 0` -> NaN.
+- `Inf * finite_nonzero` -> signed Inf.
+- `0 * finite` -> signed zero.
+- Sign bit is XOR of input signs for non-NaN outcomes.
 
-The FSM is controlled by:
-- `busy` (operation in progress)
-- `counter` (stage number 1..7)
+### Rounding
+- Use round-to-nearest-even for mantissa rounding.
 
-All stage actions are performed inside a single sequential always block using `case(counter)`.
+### Overflow / underflow
+- Overflow -> signed infinity.
+- Strong underflow may become signed zero.
 
-### Stage 1 — Unpack
-- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
-- Convert biased exponent into unbiased form: `exp - 127`.
-- Capture signs.
+---
 
-### Stage 2 — Special classification + denormal setup
-- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
-- For normal operation:
-  - If exponent is nonzero => sets implicit leading 1: `a_m[23] = 1`.
-  - If exponent is zero (subnormal) => forces exponent to -126 (subnormal exponent baseline).
+## 4) Implementation guidance (not mandatory)
 
-> If you restrict inputs to **normal numbers only**, then:
-> - `expA` and `expB` are always 1..254,
-> - hidden-one insertion always happens,
-> - special logic is bypassed in practice.
+You may choose any internal structure that satisfies external behavior:
 
-### Stage 3 — Input normalization (lightweight)
-- If mantissa MSB is not set, shift left and decrement exponent.
-- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.
+- single always block + counter/FSM, or
+- staged registers, or
+- compute-result function called when retiring request.
 
-### Stage 4 — Multiply core
-- Compute result sign: `z_s = a_s ^ b_s`
-- Exponent add: `z_e = a_e + b_e + 1`
-- Mantissa product: `product = a_m * b_m * 4`
-  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.
+A common robust approach is:
+1. Decode sign/exponent/fraction
+2. Handle special classes first (NaN/Inf/Zero)
+3. Multiply significands
+4. Normalize
+5. Round (RNE)
+6. Pack final IEEE-754 bits
 
-### Stage 5 — Extract mantissa + rounding bits
-- `z_m = product[49:26]`
-- `guard_bit = product[25]`
-- `round_bit = product[24]`
-- `sticky = OR(product[23:0])`
+---
 
-### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)
-Normalize the result mantissa and apply IEEE-754 round-to-nearest-even
+## 5) What NOT to optimize for
 
-### Stage 7 — Pack
-- For normal path:
-  - Pack sign, biased exponent, fraction.
-  - If exponent indicates overflow -> output INF.
-  - If exponent indicates exact denorm boundary -> force exponent field to 0 (denormal/zero representation).
-- Asserts `out_valid` for one cycle and clears `busy`.
+- Do **not** prioritize matching an exact internal 7-stage datapath from prose.
+- Do **not** rely on accepting new requests while busy.
+- Do **not** emit multi-cycle `out_valid`.
 
 ---
 
-## Assumptions & Constraints
-- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)
+## 6) Validation checklist before submission
+
+1. Module compiles (`iverilog`/`verilator` etc.)
+2. `out_valid` is one-cycle pulse
+3. Latency from accept to `out_valid` is always 7 cycles
+4. Busy logic prevents overlapping operations
+5. At least smoke-test these vectors:
+   - `1.0 * 1.0 = 1.0`
+   - `2.0 * 0.5 = 1.0`
+   - `1.5 * 1.5 = 2.25`
+   - `Inf * 0 = NaN`
+   - `NaN * 3.0 = NaN`
 
 ---
 
-## Verification Notes
-Recommended testbench behavior for this handshake design:
-- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
-- Wait for `out_valid` before sampling `z`.
-- Generate only normal operands,
+## 7) Scope assumptions
 
----
\ No newline at end of file
+- Target is RTL-level functional correctness, not timing closure.
+- Deterministic, reproducible behavior is required.

# FP32 Multiplier Spec (Agent-Oriented)

## Goal
Implement `sources/multiply_fp32.sv` module `fmultiplier` as a synthesizable multi-cycle FP32 multiplier.

This task is graded by cocotb + Icarus. Prioritize deterministic behavior and bit-accurate finite multiplication with round-to-nearest-even (RNE).

## What matters most for pass rate
1. Correct `valid` / `out_valid` timing.
2. Correct normal finite multiply path (including RNE).
3. Correct overflow to infinity and obvious special cases.
4. Never emit accidental nonzero on severe underflow.

If you must simplify, keep finite normal-path math correct first.

## Interface + timing contract
- Inputs: `clk`, `rst`, `valid`, `a[31:0]`, `b[31:0]`
- Outputs: `z[31:0]`, `out_valid`

Rules:
- Accept only when idle and `valid==1`.
- Latch operands on acceptance.
- Ignore new `valid` while busy.
- Output exactly one-cycle `out_valid` pulse when result is ready.
- Keep fixed deterministic latency.

## Required numeric behavior
For `x`:
- `sign = x[31]`
- `exp  = x[30:23]`
- `frac = x[22:0]`

Classes:
- NaN: `exp==8'hFF && frac!=0`
- Inf: `exp==8'hFF && frac==0`
- Zero: `exp==0 && frac==0`
- Normal: `1 <= exp <= 254`
- Subnormal: `exp==0 && frac!=0`

Benchmark simplification policy:
- For this benchmark, you may treat subnormal **inputs** as zero-equivalent in the special-case path.
- This is preferred over partially-correct subnormal arithmetic.

Special-case priority:
1. NaN present -> quiet NaN (`32'h7FC00000` is fine).
2. `Inf * 0` -> quiet NaN.
3. Inf present -> signed Inf.
4. Zero present -> signed Zero.
5. Otherwise finite multiply.

## Finite multiply algorithm (use integer datapath)
Use this exact structure to avoid rounding bugs:

1. Sign:
   - `sign_z = sign_a ^ sign_b`

2. 24-bit significands:
   - Normal operand: `{1'b1, frac}`
   - If implementing full subnormals: subnormal operand `{1'b0, frac}`
   - If using simplification policy: route subnormal inputs through zero/special handling instead.

3. Unbiased exponent sum (normal-path):
   - Normal operand exponent contribution: `exp - 127`
   - `exp_sum = exp_a_unbiased + exp_b_unbiased`

4. Multiply significands:
   - `prod = sig_a * sig_b` (48-bit)

5. Pre-normalize by top bit:
   - If `prod[47]==1`, value is in `[2,4)`:
     - increment exponent by 1
     - take 24-bit candidate mantissa from `prod[47:24]` (includes hidden 1)
     - `guard=prod[23]`, `round=prod[22]`, `sticky=|prod[21:0]`
   - Else value is in `[1,2)`:
     - mantissa candidate from `prod[46:23]`
     - `guard=prod[22]`, `round=prod[21]`, `sticky=|prod[20:0]`

6. RNE increment condition:
   - `inc = guard && (round || sticky || mantissa_candidate[0])`
   - `mantissa_rounded = mantissa_candidate + inc`

7. Post-round renormalization:
   - If `mantissa_rounded` overflows 24 bits, right-shift by 1 and increment exponent.

8. Re-bias and pack:
   - `exp_biased = exp_unbiased_final + 127`
   - Overflow (`exp_biased >= 255`) -> signed Inf.
   - Underflow (`exp_biased <= 0`) -> **signed Zero**.
   - Do not emit subnormal outputs for this task unless you are sure they are correct.
   - Normal pack: `{sign_z, exp_biased[7:0], mantissa_rounded[22:0]}`

### Critical anti-bug checks
- If your output exponent field is `8'h00`, output fraction must be `23'd0` (signed zero only).
- Common failure pattern: expected `0x00000000` but design returns tiny nonzero; guard against this explicitly in pack stage.
- Common failure pattern: 1-ULP mismatch from stale/non-updated temporaries. In a clocked stage, compute with local temporaries using blocking assignments, then write final regs with nonblocking assignments.

## Practical implementation notes
- Keep internal exponent as signed (`integer` or signed vector wide enough for intermediate values).
- Keep all combinational math explicit; avoid real-number operations.
- A small FSM (5-7 stages) is fine; one result at a time is acceptable.

## Required self-check before final answer
Run at least these directed checks before finalizing:
- normal x normal random sweep
- max finite x 2.0
- min normal x 0.5
- NaN propagation
- Inf x 0
- very small normal x very small normal (expect zero in this task policy)
- one tie-to-even style case to confirm RNE LSB behavior

If any fail, fix RTL before final answer.

## Constraints
- Synthesizable SystemVerilog only.
- No SVA property/sequence syntax (Icarus compatibility).

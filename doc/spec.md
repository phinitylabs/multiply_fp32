## Implementation Clarifications (STRICT — REQUIRED FOR CORRECTNESS)

This section refines behavior to eliminate ambiguity.  
All implementations MUST follow this model consistently.

---

### 🔴 Core Rule — Single Consistent Scaling Model

Use **ONLY MSB-based normalization**.

DO NOT:
- Add `+1` to exponent artificially
- Multiply mantissa by `*4`
- Mix scaling approaches

Correct model:

    z_e = a_e + b_e
    product = a_m * b_m   // 24 x 24 → 48 bits

---

### Mantissa Representation

Operands MUST be treated as:

    a_m = {1'b1, a_frac}   // 24-bit
    b_m = {1'b1, b_frac}

---

### Product Width

    product must be 48 bits minimum

---

### 🔴 Normalization (CRITICAL)

After multiplication:

    product ∈ [1.0, 4.0)

So:

- If product[47] == 1 → value ≥ 2
  - shift right by 1
  - increment exponent

- Else → already normalized

Implementation:

    if (product[47]) begin
        product = product >> 1;
        z_e = z_e + 1;
    end

---

### Bit Extraction (STRICT)

After normalization:

    z_m        = product[46:23]   // 24 bits
    guard_bit  = product[22]
    round_bit  = product[21]
    sticky     = OR(product[20:0])

⚠️ DO NOT use fixed slicing before normalization  
⚠️ DO NOT use 49:26 or 50-bit assumptions

---

### 🔴 Rounding (RNE — MUST MATCH IEEE)

Apply ONLY after normalization:

    if (guard_bit && (round_bit || sticky || z_m[0]))
        z_m = z_m + 1

---

### Post-Rounding Normalization (ONLY IF NEEDED)

If rounding causes overflow:

    if (z_m == 24'h1000000) begin
        z_m = 24'h800000
        z_e = z_e + 1
    end

⚠️ This is the ONLY allowed second normalization

---

### Stage Mapping (STRICT)

#### Stage 4 — Multiply
- z_s = a_s ^ b_s
- z_e = a_e + b_e
- product = a_m * b_m

#### Stage 5 — Normalize + Extract
- Perform MSB-based normalization
- Extract z_m, G, R, S

#### Stage 6 — Round + Adjust
- Apply RNE rounding
- Handle rounding overflow ONLY

#### Stage 7 — Pack
- Apply bias
- Handle overflow/underflow

---

### 🔴 Underflow Rule (SIMPLIFIED)

If:

    z_e <= -126

Then:

    z = 0

(No denormals allowed)

---

### 🔴 Overflow Rule

If:

    z_e >= 128

Then:

    z = INF

---

### 🔴 Hidden Bit Rule

Final mantissa must drop hidden bit:

    fraction = z_m[22:0]

---

### 🔴 Absolute DON'Ts (Common Failure Causes)

DO NOT:
- Mix `*4` scaling with MSB normalization
- Slice product before normalization
- Round before normalization
- Normalize more than once (except rounding overflow)
- Use inconsistent bit ranges across stages

---

## ✅ Resulting Behavior

This guarantees:

- Deterministic rounding
- Correct normalization
- Bit-accurate IEEE-like results for normal inputs
- Consistent implementation across agents

Expected pass rate improvement:

➡️ From unstable (<40%)  
➡️ To stable **60–75% range**

---

## Verification Alignment (IMPORTANT)

Testbench MUST:

- Wait for `out_valid`
- Compare full 32-bit result
- Use only normal FP32 inputs
- Allow 1-cycle valid pulse only

---

## FSM / Pipeline Stages (Detailed)

All stage actions are performed inside a single sequential always block using `case(counter)`.

### Stage 1 — Unpack
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form: `exp - 127`.
- Capture signs.

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- For normal operation:
  - **CRITICAL:** Use 24-bit registers for `a_m` and `b_m`.
  - Explicitly prepend the hidden bit: `a_m = {1'b1, a_frac}` and `b_m = {1'b1, b_frac}`.
  - This 24-bit significand is what must be passed to the Stage 4 multiplier.

> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.

### Stage 3 — Input normalization (lightweight)
- If mantissa MSB is not set, shift left and decrement exponent.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply
- Compute result sign: `z_s = a_s ^ b_s`.
- Exponent add: `z_e = a_e + b_e` (no extra `+1` for mantissa scaling).
- Mantissa product: `product = a_m * b_m` (**48-bit minimum**; **no `* 4`**).

### Stage 5 — Normalize + Extract
- MSB-based normalization as in **Normalization (CRITICAL)** (`product` in `[1,4)`): if `product[47]`, shift right once and increment `z_e`.
- Extract `z_m`, `guard_bit`, `round_bit`, and `sticky` as in **Bit Extraction (STRICT)** (`z_m = product[46:23]`, etc.).

### Stage 6 — Round + Adjust
- Apply **Rounding (RNE)** after normalization only.
- **Post-Rounding Normalization** only if `z_m` overflows to `24'h1000000` after increment.

### Stage 7 — Pack
- For normal path: pack sign, biased exponent (+127), and fraction (`z_m[22:0]` per **Hidden Bit Rule**).
- Apply **Overflow Rule** / **Underflow Rule** (simplified) as documented above.
- Assert `out_valid` for one cycle and clear `busy`.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands.

### Additional Clarifications for Correctness

#### Normalization and Rounding Order
- Mantissa normalization MUST be completed before rounding.
- Rounding applies only after the mantissa is in normalized form.
- Guard, round, and sticky bits must be derived from the normalized product.
- Do not perform rounding on unnormalized mantissa values.

#### Single Normalization Rule
- Perform normalization exactly once before rounding (MSB shift for `product[47]` only).
- After rounding, only adjust if mantissa overflows per **Post-Rounding Normalization**.
- Do not perform multiple normalization passes beyond that.

#### Underflow Handling
- Follow **Underflow Rule**: if `z_e <= -126` in the simplified model, `z = 0` (no denormals).

#### Overflow Handling
- Follow **Overflow Rule**: if `z_e >= 128`, `z = INF`.

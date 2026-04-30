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

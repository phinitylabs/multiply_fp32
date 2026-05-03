# fmultiplier — FP32 Multiplier (Agent-Optimized Spec)

## Overview
Implement a 32-bit floating-point multiplier:

    z = a * b

Constraints:
- IEEE-754 binary32
- Round-to-nearest-even (RNE)
- Fixed latency
- Handshake-based (valid / out_valid)

---

## IMPORTANT SIMPLIFICATIONS (DO NOT VIOLATE)

- Inputs are ALWAYS normal numbers
- DO NOT handle NaN / Inf / zero / subnormal
- Output underflow MUST flush to zero
- Only one operation at a time (no pipelining)

---

## Interface

| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| clk   | in | 1 | Clock |
| rst   | in | 1 | Async reset (posedge) |
| valid | in | 1 | Start pulse (1 cycle) |
| a     | in | 32 | Operand A |
| b     | in | 32 | Operand B |
| z     | out | 32 | Result |
| out_valid | out | 1 | Result valid pulse (1 cycle) |

---

## Handshake Rules (STRICT)

Start condition:
    valid == 1 AND busy == 0

On start (cycle T):
    a_r <= a
    b_r <= b
    busy <= 1
    counter <= 1

While busy == 1:
    ignore valid

Completion (cycle T+6):
    z updated
    out_valid = 1 (exactly 1 cycle)
    busy = 0

---

## Timing (EXACT)

| Cycle | counter |
|------|--------|
| T    | 1 |
| T+1  | 2 |
| T+2  | 3 |
| T+3  | 4 |
| T+4  | 5 |
| T+5  | 6 |
| T+6  | 7 (out_valid = 1) |

Latency = EXACTLY 7 cycles

---

## Data Format

For each operand:
- sign = bit[31]
- exponent = bits[30:23]
- fraction = bits[22:0]

Derived:
- unbiased exponent = exp - 127
- mantissa = {1'b1, fraction}  (24 bits)

---

## Computation (NO AMBIGUITY)

### Step 1 — Sign
    z_s = a_s ^ b_s

---

### Step 2 — Exponent
    z_e = a_e + b_e

---

### Step 3 — Multiply

    product = a_m * b_m   // 48-bit

Range:
    product ∈ [1.0, 4.0)

---

### Step 4 — Normalize + Extract (FIXED LOGIC)

IF product[47] == 1:
    // product >= 2
    z_m = product[47:24]
    z_e = z_e + 1
    guard  = product[23]
    round  = product[22]
    sticky = OR(product[21:0])

ELSE:
    // product < 2
    z_m = product[46:23]
    guard  = product[22]
    round  = product[21]
    sticky = OR(product[20:0])

---

### Step 5 — Round to Nearest Even (RNE)

Increment if:

    guard == 1 AND (round == 1 OR sticky == 1 OR z_m[0] == 1)

If increment causes overflow:
    z_m >>= 1
    z_e += 1

---

### Step 6 — Pack

IF z_e > 127:
    z = {z_s, 8'hFF, 23'b0}   // overflow → INF

ELSE IF z_e < -126:
    z = 32'b0                 // underflow → zero

ELSE:
    exp_out = z_e + 127
    frac_out = z_m[22:0]
    z = {z_s, exp_out, frac_out}

---

## Golden Rules

1. out_valid MUST assert exactly at cycle T+6
2. out_valid MUST be 1 cycle only
3. z MUST be bit-exact IEEE result (RNE)
4. Same input → same output always

---

## Example Cases

1.0 * 1.0 = 1.0  
2.0 * 0.5 = 1.0  
1.5 * 1.5 = 2.25  

Hex:
- 1.0  = 0x3F800000
- 2.25 = 0x40100000
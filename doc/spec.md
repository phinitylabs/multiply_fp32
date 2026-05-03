# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
fmultiplier is a multi-cycle single-precision floating-point multiplier using a valid/out_valid handshake. It runs a staged pipeline (FSM via counter) and outputs a 32-bit IEEE-754 result.

Target:
- Bit-accurate for normal FP32 (round-to-nearest-even)
- Deterministic latency
- z = a * b

---

## Interface

Ports:
clk        : in  (1)   Clock
rst        : in  (1)   Async reset
valid      : in  (1)   1-cycle start pulse
a          : in  (32)  Operand A
b          : in  (32)  Operand B
z          : out (32)  Result
out_valid  : out (1)   1-cycle result pulse

---

## Handshake
- valid accepted only when not busy
- inputs latched on start
- while busy, valid ignored
- out_valid pulses for 1 cycle when result ready

---

## Latency
- Fixed 7 cycles

---

## Data Model
- sign     = bit 31
- exponent = bits 30:23
- mantissa = bits 22:0

Internal:
- mantissa extended to 24-bit with hidden 1
- product = 48-bit
- guard / round / sticky bits used

---

## Implementation Clarifications

### Input Assumptions
- Only normal numbers
- exponent in [1..254]
- no zero, subnormal, inf, NaN

---

### Mantissa Handling
a_m = {1'b1, a[22:0]}
b_m = {1'b1, b[22:0]}

---

### CANONICAL DATA PATH (MANDATORY)

product = a_m * b_m

if (product[47]) begin
    product = product >> 1
    z_e = z_e + 1
end

z_m    = product[46:23]
guard  = product[22]
round  = product[21]
sticky = OR(product[20:0])

if (guard && (round || sticky || z_m[0]))
    z_m = z_m + 1

if (z_m == 24'h1000000) begin
    z_m = 24'h800000
    z_e = z_e + 1
end

---

### Stage Ownership Rules
- Normalization only in Stage 5
- Rounding only in Stage 6

---

### Underflow
- If exponent <= -126 → output 0

---

### Overflow
- If exponent >= 128 → output INF

---

## FSM / Pipeline

### Stage 1 — Unpack
- Extract sign, exponent, mantissa
- Convert exponent to unbiased

---

### Stage 2 — Setup
- Insert hidden bit
- Prepare mantissas

---

### Stage 3 — Pass-through
- No change for normal inputs

---

### Stage 4 — Multiply
- z_s = a_s ^ b_s
- z_e = a_e + b_e
- product = a_m * b_m (48-bit)

---

### Stage 5 — Normalize + Extract
if (product[47]) begin
    product = product >> 1
    z_e = z_e + 1
end

z_m    = product[46:23]
guard  = product[22]
round  = product[21]
sticky = OR(product[20:0])

---

### Stage 6 — Round (RNE)
if (guard && (round || sticky || z_m[0]))
    z_m = z_m + 1

if (z_m == 24'h1000000) begin
    z_m = 24'h800000
    z_e = z_e + 1
end

---

### Stage 7 — Pack
- Apply bias (+127)
- If exponent >= 255 → INF
- If exponent <= 0 → 0
- Drop hidden bit
- Pack result
- out_valid = 1

---

## Verification Notes
- valid is 1-cycle pulse
- wait for out_valid
- compare full 32-bit result
- only normal inputs
# Relaxed SIMD proposal

## Summary

This proposal adds a set of useful SIMD instructions that introduce local
non-determinism (where the results of the instructions may vary based on
hardware support).

## Motivation

Applications running on Wasm SIMD cannot take full advantage of hardware
capabilities. There are 3 reasons:

1. Instruction depends on hardware support
2. Approximate instructions that are underspecified in hardware
3. Some SIMD instructions penalize particular architecture

See [these
slides](https://docs.google.com/presentation/d/1Qnx0nbNTRYhMONLuKyygEduCXNOv3xtWODfXfYokx1Y/edit?usp=sharing)
for more details.

## Overview

Broadly, there are three categories of instructions that fit into the Relaxed SIMD proposal:

1. Integer instructions where the inputs are interpreted differently (e.g.
   swizzle,  4-D dot-product)
2. Floating-point instructions whose behavior for out-of-range and NaNs differ
   (e.g. float-to-int conversions, float min/max)
3. Floating-point instructions where the precision or order of operations
   differ (e.g. FMA, reciprocal instructions, sum reduction)

Example of some instructions we would like to add:

- Fused Multiply Add (single rounding if hardware supports it, double rounding if not)
- Approximate reciprocal/reciprocal sqrt
- Relaxed Swizzle (implementation defined out of bounds behavior)
- Relaxed Rounding Q-format Multiplication (optional saturation)

## References

- Poll for phase 1
  [presentation](https://docs.google.com/presentation/d/1Qnx0nbNTRYhMONLuKyygEduCXNOv3xtWODfXfYokx1Y/edit?usp=sharing)
  and [meeting
  notes](https://github.com/WebAssembly/meetings/blob/master/main/2021/CG-03-16.md)
- [SIMD proposal](https://github.com/WebAssembly/simd)

## How to suggest a instructions

1. [File an issue](https://github.com/WebAssembly/relaxed-simd/issues/new)
2. In the body of the issue, copy the template below and answer the questions (an example of FMA has been given below):

```
1. What are the instructions being proposed?

2. What are the semantics of these instructions?

3. How will these instructions be implemented? Give examples for at least x86-64 and ARM64.

4. How does behavior differ across processors? What new fingerprinting surfaces will be exposed?
```

An example FMA proposal based on https://github.com/WebAssembly/simd/pull/79:

    > 1. What are the instructions being proposed?

    - `f32x4.qfma`
    - `f32x4.qfms`
    - `f64x2.qfma`
    - `f64x2.qfms`

    > 2. What are the semantics of these instructions?

    ```
    f32x4.qfma(x, y, z) = for each 32-bit lane: x[i] + y[i] * z[i] (single rounding)
    f32x4.qfms(x, y, z) = for each 32-bit lane: x[i] - y[i] * z[i] (single rounding)
    f64x2.qfma(x, y, z) = for each 64-bit lane: x[i] + y[i] * z[i] (single rounding)
    f64x2.qfms(x, y, z) = for each 64-bit lane: x[i] - y[i] * z[i] (single rounding)
    ```

    > 3. How will these instructions be implemented? Give examples for at least x86-64 and ARM64.

    ia32 or x64 with FMA3:

    ```
    f32x4.qfma = VFMADD231PS(a, b, c)
    f32x4.qfms = VFNMADD231PS(a, b, c)
    f64x2.qfma = VFMADD231PD(a, b, c)
    f64x2.qfms = VFNMADD231PD(a, b, c)
    ```

    ARM64:

    ```
    // examples
    ```

    ARMv7 with NEONv2:

    ```
    // examples
    ```

    All others:

    ```
    // examples
    ```

    > 4. How does behavior differ across processors? What new fingerprinting surfaces will be exposed?

    On hardware without FMA support, it will be calculated with two roundings. This will allow us to differentiate between
    - processor generations on Intel (newer generations come with FMA support).
    - some older 32-bit ARM (Cortex-A8, Cortex-A9, Qualcomm Scorpion).

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

### Consistency

This proposal introduces non-deterministic instructions - given the same
inputs, two calls to the same instruction can return different results. For
example:

```wast
(module
  (func (param v128 v128 v128)
    (f32x4.qfma (local.get 0) (local.get 1) (local.get 2))   ;; (1)
    ;; some other computation
    (f32x4.qfma (local.get 0) (local.get 1) (local.get 2)))) ;; (2)
```

The same instruction at `(1)` and `(2)`, when given the same inputs, can return
two different results. This is compliant as the instruction is
non-deterministic, though unlikely on certain embeddings like the Web (where
the same implementation for `f32x4.qfma` is likely to be used for all calls to
that instruction). One can imagine splitting an application's module and
running them on multiple runtimes, where the runtimes produce
different results - this can be surprising to the application.

Applications can specify their consistency needs using a new type `fpenv` (name
subject to change). A `fpenv` is defined in the `fpenv` section of a module.
All Relaxed SIMD instructions will take an additional `varuint32` immediate,
which is an index into the `fpenv` index space:

```wast
(module
  (func (param v128 v128 v128)
    (f32x4.qfma 0 (local.get 0) (local.get 1) (local.get 2)) ;; (1)
    ;; ...
    (f32x4.qfma 0 (local.get 0) (local.get 1) (local.get 2)) ;; (2)
  )
  (fpenv 0))
```

In the example above, both `f32x4.qfma` instructions refer to the same `fpenv`,
and will get the same results when given the same input.

A `fpenv` has a single `varuint32` attribute. This is a opaque value that does
not represent anything in particular. The identity of a `fpenv` is the value
encapsulated inside of it, which is non-deterministic. This `fpenv` can be
imported/exported between modules to require consistency. The attribute is
reserved for future extensibility.

```wast
;; module a
(module
  (fpenv (export "foo") 0)
  (func (param v128 v128 v128)
    (f32x4.qfma 0 (local.get 0) (local.get 1) (local.get 2)))) ;; (1)
```

```wast
;; module b
(module
  (import "a" "foo" (fpenv))
  (func (param v128 v128 v128)
    (f32x4.qfma 0 (local.get 0) (local.get 1) (local.get 2)))) ;; (2)
```

In the above example, the same `fpenv` is exported by module `a`, and imported
by module `b`, so instructions `(1)` and `(2)` will consistently return the
same results when given the same inputs.

Relaxed SIMD instructions are deterministic "up-to" `fpenv`. Given the
identical inputs, two calls to the same instruction will return the same result
provided that the `fpenv` is the same, where the identity of `fpenv` is opaque
and must be passed via import/export.

## References

- Poll for phase 1
  [presentation](https://docs.google.com/presentation/d/1Qnx0nbNTRYhMONLuKyygEduCXNOv3xtWODfXfYokx1Y/edit?usp=sharing)
  and [meeting
  notes](https://github.com/WebAssembly/meetings/blob/master/main/2021/CG-03-16.md)
- [SIMD proposal](https://github.com/WebAssembly/simd)

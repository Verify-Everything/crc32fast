# Verified fork of crc32fast 1.5.3

This branch is upstream crc32fast 1.5.3 (commit d2363eab) with two small patches. Every function
is proved in Lean 4 (through Aeneas) to equal a bit-at-a-time reference CRC-32 for every input.

## Patches
1. **Hasher: wrap the byte count instead of panicking on overflow.** Upstream's
   `self.amount += len` panics in debug builds when the `u64` length overflows (for example
   `Hasher::new_with_initial_len(0, u64::MAX)` then `update`). Release builds already wrap.
   This uses `wrapping_add` in `update` and `combine`. The CRC value is unchanged.
2. **Make the SIMD kernels readable by Aeneas.** The x86 unaligned loads become `load128/256/512`
   helpers, and the aarch64 index loops and `align_to` call become plain loops and `split_quads`.
   No behaviour change. Tested against upstream on every kernel.

## What is proved
- Builds: riscv64 no_std (portable), x86_64 no_std and std (SSE, AVX2 and AVX-512 VPCLMULQDQ
  kernels), aarch64 no_std and std (CRC32B/CRC32X kernel, including the 3-stream path).
- `hash`, `Hasher::{new, new_with_initial, new_with_initial_len, update, finalize, reset, combine,
  total_len}` and every internal function: equal to the reference CRC-32 and never panic.
  Incremental updates and `combine` equal one pass over the concatenation.
- Only Lean's standard axioms (`propext`, `Classical.choice`, `Quot.sound`).

## Trusted base
Aeneas/Charon's translation, Lean's kernel, and hand-written Lean models of the CPU instructions
and a few `core` functions. The instruction models are checked against real CPUs (x86) and qemu
(aarch64) on random inputs. One documented assumption: `_mm256_castsi128_si256` and
`_mm512_castsi128_si512` give zero upper bits (Intel leaves them undefined; Rust fills zeros).

Proofs and the pipeline that re-checks them: https://github.com/Verify-Everything/verified-libs

# 0003 — LZ4 ARMv8 NEON decompression — REVERTED

**Status: attempted, measured, reverted.** The stack below was applied and built
successfully (including the NEON assembly), but the ABI baseline gate rejected it:
**13 exported symbols changed CRC**, so the kernel was no longer module-compatible
with the stock `vendor_dlkm`, which is this project's core invariant.

Cause is structural, not a declaration mismatch. This vendoring replaces the
kernel's LZ4 fork with upstream lz4 1.10, and the two generations lay out their
stream state differently:

| | kernel fork | upstream lz4 1.10 |
|---|---|---|
| stream state | `uint32_t hashTable[LZ4_HASH_SIZE_U32]` inline | external table pointer plus `tableType` / `dictCtx` |

Every export whose type reaches `LZ4_stream_t *`, `LZ4_streamHC_t *` or
`LZ4_streamDecode_t *` therefore gets a new CRC: `LZ4_compress_default`,
`LZ4_compress_fast`, `LZ4_compress_HC`, `LZ4_loadDict`, `LZ4_loadDictHC`,
`LZ4_saveDict`, `LZ4_saveDictHC`, `LZ4_resetStreamHC`, `LZ4_setStreamDecode`,
`LZ4_compress_fast_continue`, `LZ4_compress_HC_continue`,
`LZ4_decompress_fast_continue`, `LZ4_decompress_safe_continue`.

An ABI-safe alternative exists — keep the kernel's `lib/lz4` intact and add only
the NEON fast path plus a resume-capable decoder entry — but the real win is the
EROFS read path, which this stack does not cover at all (see "Not covered: EROFS"
below). The notes are kept for whoever picks it up.

## Original attempt

Applied to `android_kernel_motorola_blanc` only; no other Kokuban project uses it.

## Source

| | |
|---|---|
| Repo | [`zzh20188/GKI_KernelSU_SUSFS`](https://github.com/zzh20188/GKI_KernelSU_SUSFS), branch `dev` |
| Files | `zram/lz4/*`, `zram/include/linux/lz4.h`, `zram/apply_lz4_neon.sh` |
| Driver | `.github/workflows/build.yml`, step `配置 ZRAM LZ4 补丁栈` (`if: inputs.use_zram`) |
| Origin | `lib/lz4/lz4armv8/lz4armv8.S` is the Huawei EROFS LZ4 NEON decompressor (`_lz4_decompress_asm`) |

`apply_lz4_neon.sh` is **not** self-contained: on its own it rewrites four call
sites to call `LZ4_arm64_decompress_safe()`, which does not exist in the stock
6.12 tree. It only works together with the `zram/lz4/*` library replacement that
defines and exports it. Both halves are applied here.

## What changed in the tree

`lib/lz4/` is replaced by the upstream LZ4 library, **adapted to the kernel API**:
`LZ4_compress_default()` and friends keep their `void *wrkmem` parameter, and the
kernel-exported symbol set is preserved. Deleted: `lz4_compress.c`,
`lz4_decompress.c`, `lz4hc_compress.c`, `lz4defs.h` (nothing outside `lib/lz4/`
included `lz4defs.h`).

Added: `lib/lz4/lz4armv8/lz4armv8.S` (the NEON block decoder), `lz4accel.c/.h`
(per-CPU entry selection, `kernel_neon_begin/end`, a Cortex-A53 variant without
PRFM, and a `CONFIG_CFI_CLANG` trampoline — this device builds with CFI on, so
that trampoline is load-bearing).

`lib/lz4/Makefile` compiles the library with `-O3 -DLZ4_FREESTANDING=1
-DLZ4_FAST_DEC_LOOP=1` and adds the armv8 objects under `CONFIG_ARM64`.

`include/linux/lz4.h` becomes a forwarder to the vendored headers. The four call
sites below switch to the NEON entry point under
`#if defined(CONFIG_ARM64) && defined(CONFIG_KERNEL_MODE_NEON)`, and are otherwise
unchanged:

| Call site | Consumer |
|---|---|
| `crypto/lz4.c` | the `lz4` crypto algorithm, i.e. **zram** (zram is a stock module that allocates `lz4` through the crypto API) |
| `crypto/lz4hc.c` | the `lz4hc` algorithm (`CONFIG_CRYPTO_LZ4HC` is off here) |
| `fs/f2fs/compress.c` | F2FS compressed-file reads |
| `fs/incfs/data_mgmt.c` | Incremental FS |

`dip` is `false` at all four sites: the source and destination buffers do not
overlap. The asm's `Done1` path (`cbz x5, Done`) restores the source pointer by
one byte when `dip` is set, which is what in-place (overlapping) decoding needs —
see the EROFS note below.

## Not covered: EROFS

`fs/erofs/decompressor.c` calls `LZ4_decompress_safe_partial()` /
`LZ4_decompress_safe()` directly, so **the system/vendor read path does not get
NEON** from this patch even though it is the largest consumer of LZ4 on Android.
Adding it means passing `dip = (maptype == 3)`, because `maptype == 3` is EROFS's
in-place mode (`rq->inplace_io` with `omargin >=
LZ4_DECOMPRESS_INPLACE_MARGIN()`). That is deliberately left out until it can be
tested: a wrong `dip` corrupts reads silently instead of failing loudly.

## Verifying

The ABI baseline gate grades every build, so a CRC or export change here fails the
build. Expected result: 0 CRC mismatch, 0 missing, plus two new harmless exports
(`LZ4_arm64_decompress_safe`, `LZ4_arm64_decompress_safe_partial`).

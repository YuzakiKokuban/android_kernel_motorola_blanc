# 0005 — LZ4 ARMv8 NEON decompression, ABI-safe

Applied to `android_kernel_motorola_blanc` only.

## Why not 0003

`0003-lz4-neon.md` vendored upstream lz4 1.10 over the kernel's LZ4 fork. It built
(including the NEON assembly) but the ABI gate rejected it: **13 exported symbols
changed CRC**, because the two generations lay out stream state differently (the
kernel keeps `hashTable[LZ4_HASH_SIZE_U32]` inline, upstream keeps an external
table pointer plus `tableType`/`dictCtx`). That breaks the stock `vendor_dlkm`,
which is this project's core invariant.

Scanning the stock modules settled who actually cares:

| module | imports LZ4? |
|---|---|
| `zram.ko` | **no** — it goes through the crypto API (`crypto_comp_*`) |
| `moto_swap5.ko` | **yes** — 10 symbols, 9 of them with a changed CRC |

`moto_swap5.ko` is Motorola's own compressed-swap / memory-extension driver
(zsmalloc + LZ4 HC + `swp_swapcount`, bypassing zram), it is listed in
`modules.load`, and there is no source for it in the Moto tree — so it cannot be
rebuilt and cannot be turned into a built-in. It has to keep loading.

The ABI gate constrains symbol **types**, not function **bodies**. So this change
leaves every type and every export exactly as it was and only makes the insides
faster. **Zero CRC changes, zero missing, zero extra exports.**

## What changed

| File | Change |
|---|---|
| `lib/lz4/lz4armv8/lz4armv8.S` | the ARMv8 NEON decoder, taken verbatim from the Huawei EROFS patch |
| `lib/lz4/lz4armv8/lz4accel.c` | per-CPU dispatch, plus `noprfm` variant for Cortex-A53, plus the CFI trampolines the indirect call needs |
| `lib/lz4/lz4armv8/lz4accel.h` | wrapper around `kernel_neon_begin()`/`end()`, `LZ4_FAST_MARGIN = 128` |
| `lib/lz4/lz4armv8/Makefile` | builds the two objects |
| `lib/lz4/Makefile` | adds the subdirectory when `CONFIG_ARM64 && CONFIG_KERNEL_MODE_NEON` |
| `lib/lz4/lz4_decompress.c` | `LZ4_decompress_generic()` gains a resume point; `LZ4_decompress_safe()` and `LZ4_decompress_safe_partial()` run the assembly first |
| `include/linux/lz4.h` | declares the two dip-aware entry points |
| `fs/erofs/decompressor.c` | passes `dip = (maptype == 3)` |

## How the handover works

The assembly decodes complete LZ4 sequences until it would run past
`dst_end = dest + outputSize - LZ4_FAST_MARGIN`, saving the (source, destination)
pair at every token boundary, and returns those pointers. The generic C decoder
then resumes from that point. `iend`/`oend` are still computed from the original
buffer starts, so the resumed decode sees the real ends.

`LZ4_FAST_MARGIN` is the slack the assembly's wild copies may write past their
current output position; stopping 128 bytes early keeps them inside the buffer.

### `dip`

`Done1` in the assembly is:

```asm
	cbz	x5, Done            /* dip == 0: hand over as is */
	sub	save_src, offset_src_ptr, #1
	strb	w_tmp_match_length, [save_src]   /* dip != 0: rewrite the token */
	add	save_dst, save_dst, literal_length
```

With `dip == 0` the assembly stops at the token boundary and **never writes to the
source**. It reports failure only for a malformed stream. With `dip != 0` it
rewrites the partially consumed token in the source, which is required when the
output has already overwritten the input — EROFS `maptype 3`.

## Two safety properties

1. **Overlap is checked, not assumed.** `LZ4_decompress_safe()` and
   `LZ4_decompress_safe_partial()` only use the accelerator when the buffers do
   not overlap. EROFS decodes in place for `maptype 3` and calls the dip-aware
   entry points directly instead.
2. **Failure falls back to a plain decode.** `lz4_accel_prefix()` leaves the
   resume pointers at the buffer start when it declines, so the generic decoder
   redoes the whole block the unaccelerated way. For any input the assembly does
   not like, behaviour is identical to a kernel built without it.

Both hold because the general entry points always run with `dip == false`, so the
source is still pristine when the fallback happens.

## What benefits

`LZ4_decompress_safe_continue()` already calls `LZ4_decompress_safe()` on its
first block, so accelerating the latter covers, with no call-site change:

* `moto_swap5.ko` — the reason this is ABI-safe at all
* zram, through `crypto/lz4.c` (`CONFIG_CRYPTO_LZ4=y`, `CONFIG_ZRAM=m`)
* F2FS compression and incFS installs
* EROFS, for every maptype including in-place

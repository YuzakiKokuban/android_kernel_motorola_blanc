# Motorola Razr Fold 2026 (`blanc`) — Kokuban kernel tree

Single-tree kernel source for **Kokuban Kernel CI Center** project `razrfold_sm8845`.

| | |
|---|---|
| Device | Motorola Razr Fold 2026, codename **`blanc`** (product `blanc_g`) |
| SoC | Snapdragon **SM8845 / canoe** |
| Android build | `motorola/blanc_g/blanc:16/W3WB36.36-48-5` |
| Kernel | GKI **6.12.38-android16-5** |
| Upstream | [`MotorolaMobilityLLC/kernel-common`](https://github.com/MotorolaMobilityLLC/kernel-common) tag `MMI-W3WB36.36-48-5` |
| Branch | `main` |

This repo is a fork of Motorola's published `kernel-common` at the immutable build
tag. Only the GKI `common` kernel is built here — **no vendor modules** — because
the stock `vendor_dlkm` must stay on the device (see *ABI* below).

## Why the config is not just the stock one

`arch/arm64/configs/blanc_defconfig` is the config dumped from the shipping build
(`/proc/config.gz`) with three fixes that a from-source build *requires*:

| Setting | Stock | Here | Why |
|---|---|---|---|
| `CONFIG_GKI_TASK_STRUCT_VENDOR_SIZE_MAX` | `1024` | **`512`** | Stock *reports* 1024 but ships `u64[64]`, i.e. 512. Building at 1024 changes the `vendor_data_pad` CRC to `0xa4519653`, and `sched-walt`, `cpu_mpam`, `hung_task_enh`, `minidump` and `qca_cld3_peach_v2` then refuse to load → `Kernel panic - not syncing: Attempted to kill init!` |
| `CONFIG_TRIM_UNUSED_KSYMS` | `y` | **`n`** | An untrimmed kernel exports every symbol, which is what out-of-tree modules (Hybrid Mount, EVDI-style drivers) need. |
| `CONFIG_MODULE_SIG_PROTECT_LIST` | `"protected_module_names_list"` | **`""`** | Otherwise GKI refuses to load the stock Google-signed `rfkill`/`bluetooth`/`nfc` modules against our vmlinux: `rfkill_alloc` is rejected as a protected export, `btfmcodec_dev` never registers, the sound card never comes up and `sys.boot_completed` never flips. |

`CONFIG_GKI_TASK_STRUCT_VENDOR_SIZE_MAX=512` is the one that matters most: it is
what makes the from-source `vmlinux` **ABI-identical to the shipped kernel**. CI
enforces the ABI with `abi_symbol_gates` in `configs/projects.json` — the CRCs of
`module_layout` (the aggregate GKI ABI canary), `vendor_data_pad`, `init_task`,
`__put_task_struct`, `init_uts_ns`, `init_user_ns` and `sched_setscheduler` must all
match the verified-good values, so a struct-layout or config drift fails the build
instead of producing a bootlooping image.

## Game tuning

The tuning delta lives in [`Kokuban/tuning.fragment`](tuning.fragment) and is
overlaid onto the defconfig by CI just before configuration, so the 16-line
fragment — not an 8000-line `.config` — is the reviewable source of truth.

* **TCP BBR and TCP Brutal** (`net/ipv4/tcp_brutal.c`) are compiled in for
  lower-latency, lower-jitter online play. The system default stays `cubic`; opt in
  per device with `sysctl net.ipv4.tcp_congestion_control=bbr`.
* **ADIOS** (`block/adios.c`, Adaptive Deadline I/O Scheduler, ported from
  [`cctv18`](https://github.com/cctv18)) is built in and selected as the default
  multi-queue scheduler, which cuts interactive I/O latency under load.

Both are built **into** the kernel (`=y`). This pipeline builds `Image` only, so a
`=m` module would never be built and could not take effect.

Ported patches are kept verbatim under `Kokuban/patches/` for provenance.

Deliberately **not** touched: `CONFIG_HZ` (stock 250), the default congestion
control, THP mode, and every vendor driver — the GPU, display, touch, thermal and
audio stacks live in the stock `vendor_dlkm` that this kernel does not replace, so
kernel-side game tuning on this device is limited to the core kernel.

## ABI

The stock, Google-signed vendor modules are never rebuilt, so the built kernel must
keep every previously exported symbol's CRC. The tuning above only *adds* built-in
code, so it does not shift existing CRCs. Anything that changes a struct layout
touches the GKI ABI and will bootloop the device — check the CRC first:

```sh
grep -w vendor_data_pad out/Module.symvers     # must be 0xf54e5881
```

### Verified

The baseline `main` build was compared against a known-good from-source kernel for
this device: **16,516 non-Rust symbols exported by `vmlinux`, identical symbol set,
0 CRC differences** — including `module_layout` (`0xe976b219`, the aggregate GKI ABI
hash), `init_task` (`0x35075030`) and `vendor_data_pad` (`0xf54e5881`). The only
churn is Rust mangled names, whose crate hash changes on every build and which no C
vendor module consumes. The `ReSukiSU + SuSFS + Hybrid Mount` build reproduces the
same gate, so those additions are ABI-neutral too.

## Flashing

Flash **only** `boot`; keep `vendor_dlkm`, `system_dlkm`, `vendor_boot`, `dtbo` and
`vbmeta` stock. Flashing the from-source `vendor_dlkm` deletes the proprietary
modules and the device will not boot.

Kokuban ships an AnyKernel3 zip (`RazrFold_Kernel-*`), which replaces the kernel
inside `boot` and leaves every other partition alone. A device-checked flash
requires `ro.product.device` = `blanc`.

## Building

CI entry point: **Kokuban Kernel CI Center → Build Kernel → project
`razrfold_sm8845`**, branch `main` (LKM) or `resukisu` (ReSUKiSU + SuSFS + Hybrid
Mount). The tree is configured with `make blanc_defconfig` and built with the ACK
clang toolchain; no Bazel/`repo` manifest is involved.

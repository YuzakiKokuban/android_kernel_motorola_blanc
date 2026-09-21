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

Deliberately **not** touched by the default profile: the default congestion
control, THP mode, and every vendor driver — the GPU, display, touch, thermal and
audio stacks live in the stock `vendor_dlkm` that this kernel does not replace, so
kernel-side tuning on this device is limited to the core kernel.

### Tuning stages, measured

Every stage under `Kokuban/tuning/` was built on its own in CI and graded against
`Kokuban/abi/vmlinux.symvers.golden` (16,516 non-Rust `vmlinux` exports). "Clean"
means zero CRC differences and zero symbols lost.

| Stage | Change | Measured against the baseline | Adopted |
|---|---|---|---|
| `00-base` | ADIOS default I/O scheduler; TCP BBR + TCP Brutal | clean | yes |
| `10-network` | CAKE / FQ_PIE / PIE / SFB / RED; drops the obsolete `=m` congestion controls | clean (3 new `pie_*` exports) | yes |
| `30-hz` | `CONFIG_HZ` 250 → 300 | clean | yes |
| `31-preempt-dynamic` | `PREEMPT_DYNAMIC` | clean | yes |
| `32-sched-accounting` | `SCHEDSTATS` / `SCHED_INFO` off | **10,619 of 16,516 CRCs changed** | **no** |
| `20-slub-debug-off` | `SLUB_DEBUG` off | 4 symbols lost (its own: `get_each_kmemcache_object`, `get_slabinfo`, `get_track`, `validate_slab_cache`) | no |
| `21-init-off` | `KFENCE` + `INIT_ON_ALLOC` + `INIT_STACK_ALL_ZERO` off | `init_on_alloc` CRC changed + `__kfence_pool`, `kfence_sample_interval` lost | no |
| `40-kasan-off` | `KASAN` + `KASAN_HW_TAGS` off | 5 symbols lost (its own: `__kasan_kmalloc`, `kasan_flag_enabled`, `kasan_flag_vmalloc`, `kasan_mode`, `mte_async_or_asymm_mode`) | no |
| LTO | `--lto thin` | **16,447 of 16,516 CRCs changed, 69 symbols lost** | **no** |

Two results are worth stating plainly:

* **`SCHED_INFO` is embedded in `struct task_struct`.** Turning it off shrinks the
  struct, so every symbol whose type reaches `task_struct` changes CRC — 64 % of the
  whole ABI from one config line. This is the single most dangerous knob in the list.
* **Thin LTO reshapes essentially the entire ABI.** Inlining and symbol elimination
  move 99.6 % of CRCs and drop 69 exports, so a GKI kernel that must load stock
  modules cannot be built with LTO at all.

The three debug-off stages are a different kind of result: they lose only their own
feature's symbols, which no vendor driver has any reason to import. That cannot be
proven without the stock modules, so they stay out of the default and are offered as
`Kokuban/profiles/candidate-debug-off.fragment`, which documents the exact
`allowed_missing_symbols` / `allowed_crc_symbols` list needed to accept them. That
profile is worth a flash test, not a default.

The default `Kokuban/tuning.fragment` therefore adopts `00-base`, `10-network`,
`30-hz` and `31-preempt-dynamic`, and every build re-checks that.

### Feature stages

Not every fragment in `Kokuban/tuning/` is a tuning — the directory is the
project's kconfig entry point.

| Stage | Change | Notes |
|---|---|---|
| `90-ntsync` | `CONFIG_NTSYNC=y` | Enables `/dev/ntsync` for the Wine/Proton game layer. 6.12 ships only the first revision of that driver (semaphores, 247 lines) behind `depends on BROKEN`, so it is replaced with the complete driver. See `Kokuban/patches/0004-ntsync.md`. |

To grade any other stage or a new one:

```
Build Kernel → project razrfold_sm8845
  kconfig_fragment: Kokuban/profiles/probe-kasan.fragment
  lto:              thin            # optional
```

## Scheduling

Four independent layers, cheapest first.

**1. Runtime, no rebuild — sched_ext.** The kernel has `CONFIG_SCHED_CLASS_EXT=y`
with `BPF_SYSCALL`, `BPF_JIT`, `BPF_JIT_ALWAYS_ON`, `DEBUG_INFO_BTF` and
`SCHED_CORE` off, so `/sys/kernel/sched_ext/` exists and the fair class can be
replaced by a BPF scheduler at run time (`state`, `ops`, `switch_all`). Two caveats
must be checked on hardware: SELinux may refuse the BPF program load, and Moto
drives frequency through vendor hooks + the stock `sched-walt` module, which may
disagree with a scx scheduler's placement.

**2. Runtime knobs.** With `CONFIG_SCHED_DEBUG=y` (kept on purpose, see stage 30)
the classic tunables are available: `/proc/sys/kernel/sched_*`, EEVDF's
`/sys/kernel/debug/sched/base_slice_ns`, and the `sched_feat` toggles. cgroup
`cpu.uclamp.min/max` and cpusets pin a game to the prime cores; ADIOS is already the
default block scheduler.

**3. Config (stage 30).** `HZ=300` trades a little timer overhead for finer
scheduling granularity; `PREEMPT_DYNAMIC` allows switching preemption mode at run
time via `/sys/kernel/debug/sched/preempt`; `SCHEDSTATS`/`SCHED_INFO` are pure
per-task accounting and cost nothing to drop. Already good in the stock config:
`PREEMPT=y`, `UCLAMP_TASK` + `UCLAMP_TASK_GROUP`, `LRU_GEN`, `PSI`, schedutil,
`SCHED_MC`, and `RCU_NOCB_CPU_DEFAULT_ALL` (RCU callbacks offloaded, less jitter).
`SCHED_AUTOGROUP` stays off: Android schedules through cgroups and Google disables
it deliberately.

**4. Source patches — not done.** EEVDF/PELT constant changes or a CPU input boost
have to be patched into `kernel/sched/`, which is exactly the class of change the
ABI baseline exists to catch. Grade any such patch with the baseline before
adopting it.

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

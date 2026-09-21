# 0004 — NTSYNC (NT synchronisation primitives, `/dev/ntsync`)

Applied to `android_kernel_motorola_blanc` only.

## Why

Wine/Proton-based Windows game layers on Android (GameNative, Winlator,
Droidspaces) need NT synchronisation primitives in the kernel. Without `/dev/ntsync`
they emulate them in user space, which is much slower and is the usual cause of
stutter in Windows titles.

## What the tree already had

`drivers/misc/ntsync.c` exists in 6.12 but is only the **first revision** of the
driver: 247 lines implementing `NTSYNC_TYPE_SEM` (semaphores) alone. Its Kconfig
entry is gated behind `depends on BROKEN`, so stock never builds it and no
userspace ABI was ever exposed.

## What this change does

| File | Change |
|---|---|
| `drivers/misc/ntsync.c` | replaced with the complete driver (1272 lines): semaphore + mutex + event, `WAIT_ANY`/`WAIT_ALL` |
| `include/uapi/linux/ntsync.h` | replaced with the full UAPI (mutex/event/wait structures and ioctls) |
| `drivers/misc/Kconfig` | `depends on BROKEN` → `default m`, so the symbol is selectable |
| `Kokuban/tuning/90-ntsync.fragment` | `CONFIG_NTSYNC=y` |

Built in, not modular: this pipeline compiles `Image` only, so `=m` would produce
nothing.

Sources, as used by `zzh20188/GKI_KernelSU_SUSFS`:

* `https://github.com/Goldzxcbug/Droidspaces_Kernel_patch` → `NTsync/ntsync_base.patch`
  (driver + UAPI) and `NTsync/ntsync_compat_android16-6.12.patch` (the Kconfig
  dependency). The base patch adds those two files outright, so it cannot be
  applied here where the older revision already exists; the resulting files are
  committed instead.

## Notes

* No `EXPORT_SYMBOL` anywhere in the driver, so the module ABI is untouched; the
  baseline gate is expected to report 0 CRC mismatch and 0 missing.
* The driver's init sets `/dev/ntsync` to mode `0666` and SELinux context
  `u:object_r:gpu_device:s0`, which is what lets an unprivileged game process open
  it without a vendor SELinux policy change.
* Verify on hardware with `ls -lZ /dev/ntsync` and by running a title that needs
  NT synchronisation.

# SPDX-License-Identifier: GPL-2.0-only
#
# ARMv8 NEON LZ4 decompression. Built only when lib/lz4/Makefile decides the
# accelerator is usable (CONFIG_ARM64 && CONFIG_KERNEL_MODE_NEON), which is the
# same condition lz4accel.h gates its declarations on.
ccflags-y += -O3

obj-y += lz4accel.o lz4armv8.o

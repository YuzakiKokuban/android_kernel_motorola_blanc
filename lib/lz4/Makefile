# SPDX-License-Identifier: GPL-2.0-only
ccflags-y += -O3

obj-$(CONFIG_LZ4_COMPRESS) += lz4_compress.o
obj-$(CONFIG_LZ4HC_COMPRESS) += lz4hc_compress.o
obj-$(CONFIG_LZ4_DECOMPRESS) += lz4_decompress.o

# ARMv8 NEON accelerated decompression. The condition matches the one
# lz4armv8/lz4accel.h uses for its declarations; without it the header compiles
# to no-op stubs and the objects must not be built.
ifeq ($(CONFIG_ARM64)$(CONFIG_KERNEL_MODE_NEON),yy)
obj-$(CONFIG_LZ4_DECOMPRESS) += $(addprefix lz4armv8/, lz4accel.o lz4armv8.o)
endif

SELINUX_VERSION := 1.1.7.2

ifndef KP_DIR
    KP_DIR = ../KernelPatch
endif

OS_NAME = $(shell uname | tr A-Z a-z)
MACHINE = $(shell uname -m)
NDK_BIN_DIR := toolchains/llvm/prebuilt/$(OS_NAME)-$(MACHINE)/bin
ifdef ANDROID_NDK_LATEST_HOME
    NDK_PATH ?= $(ANDROID_NDK_LATEST_HOME)/$(NDK_BIN_DIR)
else ifdef ANDROID_NDK
    NDK_PATH ?= $(ANDROID_NDK)/$(NDK_BIN_DIR)
else ifdef ANDROID_NDK_HOME
    NDK_PATH ?= $(ANDROID_NDK_HOME)/$(NDK_BIN_DIR)
endif

ifdef TARGET_COMPILE
    CC := $(TARGET_COMPILE)gcc
    LD := $(TARGET_COMPILE)ld
    OBJCOPY := $(TARGET_COMPILE)objcopy
else ifdef NDK_PATH
    CC := $(NDK_PATH)/aarch64-linux-android31-clang
    LD := $(NDK_PATH)/ld.lld
    OBJCOPY := $(NDK_PATH)/llvm-objcopy
endif

# Default -O2 is safe after the freestanding-safe copy helpers and
# hook_local.data3 initialization fixes.  Keep -fno-builtin so ordinary
# C library names are never assumed; structure copies still must not
# rely on compiler-emitted memcpy (see copy_bytes and field-wise AV writes).
# Override with `make OPT_LEVEL=0` if a toolchain still miscompiles.
OPT_LEVEL ?= 2

CFLAGS = -Wall -O$(OPT_LEVEL) \
    -fno-builtin \
    -fno-tree-vectorize \
    -fno-tree-slp-vectorize \
    -fno-stack-protector \
    -fno-pic \
    -fno-pie \
    -fno-asynchronous-unwind-tables \
    -fno-unwind-tables \
    -fno-strict-aliasing \
    -ffunction-sections \
    -fdata-sections \
    -fno-common \
    -fno-exceptions \
    -DSELINUX_VERSION=\"$(SELINUX_VERSION)\"
CFLAGS += -std=gnu99

# Extra safety on clang freestanding builds: never emit libcalls for
# compiler-generated block moves if the option is available.
ifeq ($(findstring clang,$(notdir $(CC))),clang)
CFLAGS += -fno-builtin-memcpy -fno-builtin-memset -fno-builtin-memmove
endif

INCLUDE_DIRS := . include patch/include linux linux/include linux/security/selinux/include linux/arch/arm64/include linux/tools/arch/arm64/include

INCLUDE_FLAGS := $(foreach dir,$(INCLUDE_DIRS),-I$(KP_DIR)/kernel/$(dir))

objs := selinux_hook.o

all: selinux_hook_$(SELINUX_VERSION).kpm

selinux_hook_$(SELINUX_VERSION).kpm: ${objs}
	${CC} -r -nostdlib -Wl,--no-eh-frame-hdr -o $@ $^
ifneq ($(strip $(OBJCOPY)),)
	${OBJCOPY} --remove-section=.eh_frame --remove-section=.eh_frame_hdr --remove-section=.rela.eh_frame $@
endif

%.o: %.c
	${CC} $(CFLAGS) $(INCLUDE_FLAGS) -c -o $@ $<

.PHONY: clean
clean:
	rm -rf *.kpm
	find . -name "*.o" | xargs rm -f
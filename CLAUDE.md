# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

OpenIPC fork of HiSilicon's vendor U-Boot 2010.06 for Hi3536Cv100 (entry-4K NVR Cortex-A7 SoC). Imported from Hi3536C SDK V2.0.4.0 (V100R001C02SPC040). Builds via the OpenIPC HiSi U-Boot fleet's standard CI shape: build → 256 KiB partition-fit → qemu_smoke against widgetii/qemu-hisilicon's `hi3536cv100` machine → publish to `OpenIPC/firmware/latest`.

## Vendor source quirks

- **Single-target tree.** Top-level Makefile has `hi3536c_config`. CPU dir `arch/arm/cpu/hi3536c/`. Board dir `board/hi3536c/`. No per-SoC variant fan-out.
- **Naming.** Repo name follows OpenIPC's `BR2_OPENIPC_SOC_MODEL` convention (`hi3536cv100`); vendor source uses just `hi3536c`. The `hi3536c.h` config overrides `CONFIG_PRODUCTNAME` to `"hi3536cv100"` so the env's `soc` var matches what `OpenIPC/firmware`'s `hi3536cv100_lite_defconfig` sets.
- **No `compressed/` subdir** — different from `u-boot-hi3520dv200` / `u-boot-hi3519v101`. Hi3536C doesn't use the mini-boot LZMA self-extractor; the bootrom executes the wrapped `u-boot.bin + reg blob` directly.
- **Single regfile.** The vendor's `mkboot.sh` does a 3-piece `dd` dance: bytes [0..63] of u-boot.bin, then 5120 bytes of `reg_info_<soc>.bin` padded with `conv=sync`, then u-boot.bin from offset 5184 onward. CI inlines that — see `.github/workflows/build.yml`'s `Build` step.
- **Toolchain.** `arm-hisiv600-linux` (gcc 4.9.4), available as `arm-hisiv600-linux.tgz` in `OpenIPC/toolchains` v1.

## Build (manual)

```sh
make hi3536c_config
make CROSS_COMPILE=arm-hisiv600-linux- -j$(nproc)
# Then mkboot.sh-equivalent dd dance:
dd if=u-boot.bin of=fb1 bs=1 count=64
dd if=reg_info_hi3536cv100.bin of=fb2 bs=5120 conv=sync
dd if=u-boot.bin of=fb3 bs=1 skip=5184
cat fb1 fb2 fb3 > u-boot-hi3536cv100-universal.bin
```

## Source layout

- `include/configs/hi3536c.h` — vendor per-SoC config; ends with `#include <configs/hi-common.h>` to layer the OpenIPC env block on top
- `include/configs/hi-common.h` — OpenIPC convention shared header (mtdparts, OpenIPC # prompt, env helpers, CONFIG_CMD_UBI etc.); identical across the IP-camera fleet
- `arch/arm/cpu/hi3536c/` — SoC startup, DDR training, low-level init
- `board/hi3536c/board.c` — board init
- `reg_info_hi3536cv100.bin` — DDR/PLL register-init blob, prepended to u-boot.bin via the mkboot dd dance above

## OpenIPC patches (deviations from verbatim vendor source)

- **`include/configs/hi-common.h`** (new file): cloned verbatim from `u-boot-hi3519v101`. OpenIPC env block — mtdparts, bootcmd, prompts, tftp helpers, UBI/UBIFS commands.
- **`include/configs/hi3536c.h`**:
  - `CONFIG_PRODUCTNAME` → `"hi3536cv100"` (vendor was `"hi3536c"`).
  - `#undef CONFIG_SYS_MALLOC_LEN` before the include so hi-common.h's `CONFIG_ENV_SIZE + 512*1024` applies (UBI needs ≥ 512 KiB).
  - `#define CONFIG_LZO` so `lib/lzo/Makefile` actually compiles `liblzo.a` (UBIFS depends on `lzo1x_decompress_safe`).
  - `#include <configs/hi-common.h>` at the end.

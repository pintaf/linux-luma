# Fairphone 5 (QCM6490, sc7280) build inputs — alpha0

Cross-build on x86_64 with clang/LLVM/LLD (matching the FP6 recipe).
The source tree must be on a case-sensitive filesystem.

```sh
git clone -b sc7280 https://github.com/SimplyLuma/linux.git
O=../out-fp5
mkdir "$O"
```

## 1. Configuration

```sh
cp luma/fp5/alpha0/base.config "$O/.config"
scripts/kconfig/merge_config.sh -m -O "$O" "$O/.config" \
  luma/config/sc7280.fragment luma/fp5/alpha0/selinux.fragment
echo 'CONFIG_CMA_SIZE_MBYTES=384' >> "$O/.config"
make O="$O" ARCH=arm64 LLVM=1 olddefconfig
```

The family fragment carries the generic Luma options (SELinux, binder,
DMA heaps) that the base `luma.fragment` is meant to supply. The
`selinux.fragment` makes the FP5 device build explicit; boot-test it
before shipping.

The shipped `config-7.2.15-luma-fp5-alpha0` is the authoritative
configuration and is build-verified, but it has not been boot-tested on the
FP5 yet; the boot-tested lineage so far is the 7.2.9 tree. The merge above
brings the defconfig + fragment close to the shipped config but does not
reproduce it exactly: kconfig fills in the defaults behind options the
tested lineage carries explicitly, so a few knobs flip. When the difference
matters, copy the shipped config straight to `$O/.config` (host-derived
symbols such as `CONFIG_CC_VERSION_TEXT` and the `CC_CAN_LINK` probes depend
on the toolchain; `make olddefconfig` after copying adjusts those).

FP5 specifics carried by the shipped config but not the family fragment:
384 MB CMA (largest reservation the FP5 memory map accepts; 512 fails),
`CONFIG_ATH11K_AHB=m` (built-in bootloops; loaded after the WPSS remoteproc),
`CONFIG_FW_LOADER_COMPRESS_XZ=y` (compressed a660_gmu firmware).

## 2. Build

```sh
K="make O=$O ARCH=arm64 LLVM=1 \
  LOCALVERSION=-luma-fp5-alpha0 \
  KBUILD_BUILD_USER=luma KBUILD_BUILD_HOST=kernel-cross \
  KBUILD_BUILD_TIMESTAMP='2026-11-11 00:00:00 UTC' KBUILD_BUILD_VERSION=1"

$K -j4 Image dtbs modules
```

Module signing is on (`CONFIG_MODULE_SIG_ALL`); a fresh
`certs/signing_key.pem` is generated on first build.

`ath11k_ahb` and `snd-soc-sm8250` are modules; everything else the phone
needs is built in.

## 3. Device tree

Compose the bootable DTB. The FP5 ABL validates board identity from the DTB
itself (stock board is `yupik-idps-amoled`, `qcom,msm-id = <475
0x10000>, <515 0x10000>`, `qcom,board-id = <34 2>`) and the phone's `dtbo_*`
partitions are flashed blank, so the board-id overlay is applied at build
time:

```sh
dtc -@ -I dts -O dtb -o /tmp/fp5-boardid.dtbo luma/fp5/alpha0/dtb/fp5-boardid.dtso
fdtoverlay \
  -i "$O"/arch/arm64/boot/dts/qcom/qcm6490-fairphone-fp5.dtb \
  -o /tmp/fp5-composed.dtb \
  /tmp/fp5-boardid.dtbo
```

## 4. Boot image

Assemble the Android boot image v2 (page 4096, kernel + DTB + initramfs
in one image, offsets per the sc7280 `soc.toml` in SimplyLuma/luma, AVB
hash footer algorithm NONE) with the release initramfs.

RAM-boot (`fastboot boot`) on the device before any flash.

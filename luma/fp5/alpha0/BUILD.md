# Fairphone 5 (QCM6490, sc7280) build inputs — alpha0

Cross-build on x86_64 with clang/LLVM/LLD (matching the FP6 recipe):

```sh
make -C <this tree> O=$PWD/build/fp5 ARCH=arm64 LLVM=1 olddefconfig \
    KCONFIG_CONFIG=../luma/fp5/alpha0/config-7.2.9-fp5
make -C <this tree> O=$PWD/build/fp5 ARCH=arm64 LLVM=1 -j8 \
    Image drivers/net/wireless/ath/ath11k/ath11k_ahb.ko sound/soc/qcom/
```

Kernel modules are built-in (`=y`); only `ath11k_ahb.ko` and the ASoC
machine driver `snd-soc-sm8250.ko` are staged as modules.

Compose the bootable DTB (FP5 ABL validates board identity from the DTB
itself, and the phone's dtbo partitions are flashed blank, so the board-id
overlay is applied at build time):

```sh
dtc -@ -I dts -O dtb -o /tmp/fp5-boardid.dtbo luma/fp5/alpha0/dtb/fp5-boardid.dtso
fdtoverlay \
  -i arch/arm64/boot/dts/qcom/qcm6490-fairphone-fp5.dtb \
  -o /tmp/fp5-composed.dtb \
  /tmp/fp5-boardid.dtbo
```

Assemble the Android boot image v2 (page 4096, offsets per the sc7280
soc.toml in SimplyLuma/luma, AVB hash footer algorithm NONE) with the
release initramfs, then RAM-test on hardware before any flash.

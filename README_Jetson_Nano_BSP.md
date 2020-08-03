# Adapting Jetson Nano BSP for running AOSP on Antmicro Jetson Nano Baseboard

The BSP used in this example is based on the L4T 32.4.2 release.
The following instructions are targeted for a Debian 10 system; the package installation process needs to be adjusted to your specific distribution.

## Download and unpack the BSP package

```
wget https://developer.nvidia.com/embedded/L4T/r32_Release_v4.2/t210ref_release_aarch64/Tegra210_Linux_R32.4.2_aarch64.tbz2
wget https://developer.nvidia.com/embedded/L4T/r32_Release_v4.2/t210ref_release_aarch64/Tegra_Linux_Sample-Root-Filesystem_R32.4.2_aarch64.tbz2
sudo tar xf Tegra210_Linux_R32.4.2_aarch64.tbz2
sudo tar xf Tegra_Linux_Sample-Root-Filesystem_R32.4.2_aarch64.tbz2 -C Linux_for_Tegra/rootfs
cd Linux_for_Tegra
sudo ./apply_binaries.sh
```

## Kernel and device tree for the Jetson Nano

The Jetson Nano kernel and device tree are missing some crucial parts required for making use of virtualization on ARM64, so those parts need to be built from source.

1. Install dependecies

```
apt-get install git-core build-essential bc wget xxd kmod
```

2. Download cross toolchain and set up the environment

```
wget http://releases.linaro.org/components/toolchain/binaries/7.3-2018.05/aarch64-linux-gnu/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz
tar xvf gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz
export PATH=$(pwd)/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu/bin:$PATH
```

3. Fetch the kernel sources

```
git clone https://github.com/antmicro/kvm-aosp-linux.git
```

4. Configure, build and install the kernel

```
cd kvm-aosp-linux
ARCH=arm64 make tegra_defconfig
CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm64 make -j$(nproc)
sudo make modules_install ARCH=arm64 INSTALL_MOD_PATH=/path/to/Linux_for_Tegra/rootfs
sudo cp arch/arm64/boot/Image /path/to/Linux_for_Tegra/kernel/Image
sudo cp arch/arm64/boot/dts/tegra210-p3448-0002-p3449-0000-b00.dtb /path/to/Linux_for_Tegra/kernel/dtb/
```

6. Flash the target device

Put the Antmicro Jetson Nano Baseboard in recovery mode (to do it press the RST button while holding REC button), and flash the device with following commands:

```
cd /path/to/Linux_for_Tegra
sudo ./flash.sh jetson-nano-emmc mmcblk0p1
```

## Dependencies

The following instructions need to be executed on the flashed target board.

1. Install dependencies

```
sudo apt-get update
sudo apt-get install libsdl2-dev libpixman-1-dev libvirglrenderer-dev libepoxy-dev libgbm-dev qemu-kvm
git clone https://github.com/antmicro/kvm-aosp-qemu.git -b kvm-aosp
cd kvm-aosp-qemu
git submodule update --init --recursive
mkdir build && cd build
../configure --target-list=aarch64-softmmu --enable-kvm --disable-werror --enable-virglrenderer --enable-sdl --enable-opengl
make -j$(nproc)
sudo make install
cd ..
rm -rf kvm-aosp-qemu
sudo adduser nvidia kvm
```
2. Reboot the board

## Running the AOSP 10 image

To run AOSP, we need to copy the image files to the device. Building the AOSP system itself is described [in the main README file](README.md).

1. Create a directory for the image files on the device

```
mkdir ~/aosp
```

2. From the host system, copy the following AOSP files to the "aosp" directory on the device:
- out/target/product/generic_arm64/vendor.img
- out/target/product/generic_arm64/system.img
- out/target/product/generic_arm64/userdata.img
- out/target/product/generic_arm64/ramdisk.img
- out/android-4.14-q/common/arch/arm64/boot/Image

3. On the device, start Android with the following command

```
qemu-system-aarch64 \
        -enable-kvm \
        -smp 4 \
        -m 2048 \
        -cpu host \
        -M virt \
        -device virtio-gpu-pci \
        -device usb-ehci \
        -device usb-kbd \
        -device virtio-tablet-pci \
        -usb \
        -serial stdio \
        -display sdl,gl=on \
        -kernel aosp/Image \
        -initrd aosp/ramdisk.img \
        -drive index=0,if=none,id=system,file=aosp/system.img \
        -device virtio-blk-pci,drive=system \
        -drive index=1,if=none,id=vendor,file=aosp/vendor.img \
        -device virtio-blk-pci,drive=vendor \
        -drive index=2,if=none,id=userdata,file=aosp/userdata.img \
        -device virtio-blk-pci,drive=userdata \
        -full-screen \
        -append "console=ttyAMA0,38400 earlycon=pl011,0x09000000 drm.debug=0x0 rootwait rootdelay=5 androidboot.hardware=ranchu androidboot.selinux=permissive security=selinux selinux=1 androidboot.qemu.hw.mainkeys=0 androidboot.lcd.density=160"
```

It should result in the Android system booting on the device.

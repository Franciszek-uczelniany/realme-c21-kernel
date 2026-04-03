## Realme C21 Kernel

This is repo for Realme C21 (mt6765) kernel running Android 11. Note that C21Y has different SoC from different vendor.



### Building

Latest Ubuntu LTS should be fine. If you don't have ubuntu you might use docker, chroot or proot. 



#### Installing prerequisites
````
sudo apt-get update
sudo apt-get install -y bc bison build-essential ccache curl flex git \
libncurses-dev libssl-dev lld llvm make python3 unzip wget xz-utils zip cpio
# and toolchain
git clone --depth=1 https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9 -b android11-release gcc-4.9
wget https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/android10-dev/clang-r353983c.tar.gz -O clang.tar.gz
mkdir clang && tar -xf clang.tar.gz -C clang
export PATH=$(pwd)/gcc-4.9/bin:$(pwd)/clang/bin:$PATH


# Make symlink if 'python' doesn't exist, point to python3
# By default this file doesn't exist
 test ! -a /usr/bin/python && sudo ln -s /usr/bin/python3 /usr/bin/python
````





#### Building


````
cd kernel; mkdir -p out
make O=out mrproper
make O=out ARCH=arm64 oppo6765_defconfig
make O=out ARCH=arm64 olddefconfig

make -j$(nproc --all) ARCH=arm64 O=out CC=clang CROSS_COMPILE="aarch64-linux-android-" CLANG_TRIPLE=aarch64-linux-gnu-
````

### Notes

Typically, the Linux kernel decompresses its own image upon boot, but this doesn't happen on the ARM64 architecture (unlike ARM). The bootloader must decompress the Linux image, or the Linux kernel must be uncompressed. Lk on this platform supports gzip kernel compression. If you want to reduce the size of the boot image, try compressing the initramfs using xz (the default configuration currently doesn't have xz decompression enabled, but there's nothing stopping you from enabling it).

You may notice that the "optimize size" option is currently enabled in the configuration instead of "optimize speed." Remember that the Cortex-A53 cores in the MT6765 have 512kB cache across four cores, giving us 128kB L2 cache per core. Each core has 32kB L1 instruction cache and 32kB L1 data cache per core. It doesn't have L3 cache. Larger code means more cache misses. The choice of which option (-O0, -O1, or -O2) to use depends on your workload. I'll post the results in this repo when I run the benchmarks.

Phones have an SoC, which means that a single Mediatek chip integrates many other, smaller chips. Additionally, they usually have a small cache size in order to:

1) save space in the chip's housing
2) add more cores (which can be disabled or underclocked independently to save energy)
3) minimize power consumption in idle
4) reduce production costs

To flash the kernel to your device, use the magiskboot repack command. Simply copy Image.gz or Image from the arch/arm64/ directory where you placed the build result. Creating the image with mkbootimg is not recommended and may not be successful. Newer devices use a newer version of the header combined with AVB.

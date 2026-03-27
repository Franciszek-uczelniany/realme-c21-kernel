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
make mrproper
make O=out ARCH=arm64 oppo6765_defconfig
make O=out ARCH=arm64 olddefconfig
# Disable things that doesn't compile
scripts/config  --file out/.config --disable CONFIG_CFG80211
scripts/config  --file out/.config--disable CONFIG_MAC80211

echo "=== Disabling tracepoints to fix build errors ==="
scripts/config --file out/.config \
--disable CONFIG_TRACING \
--disable CONFIG_FTRACE \
--disable CONFIG_TRACEPOINTS \
--disable CONFIG_EVENT_TRACING \
--disable CONFIG_HAVE_SYSCALL_TRACEPOINTS

make -j$(nproc --all) ARCH=arm64 O=out CC=clang CROSS_COMPILE="aarch64-linux-android-" CLANG_TRIPLE=aarch64-linux-gnu-
````


#### Bugs

There is a problem linking the mac80211 module, it is most likely due to the use of incorrect header versions (from linux 4.9 vs linux 4.19). Here is a fragment of the log:


````


linux/compiler.h:193: undefined reference to `\_\_tracepoint\_cfg80211\_rx\_unexpected\_4addr\_frame'

realme-c21-kernel/gcc-4.9/bin/aarch64-linux-android-ld: net/wireless/nl80211.o: relocation R\_AARCH64\_ADR\_PREL\_PG\_HI21 against external symbol `\_\_tracepoint\_cfg80211\_rx\_unexpected\_4addr\_frame' can not be used when making a shared object; recompile with -fPIC

realme-c21-kernel/kernel/out/../include/linux/compiler.h:193: undefined reference to `\_\_tracepoint\_cfg80211\_rx\_unexpected\_4addr\_frame'

realme-c21-kernel/kernel/out/../include/linux/compiler.h:193: undefined reference to `\_\_tracepoint\_cfg80211\_rx\_unexpected\_4addr\_frame'

realme-c21-kernel/gcc-4.9/bin/aarch64-linux-android-ld: net/wireless/nl80211.o: relocation R\_AARCH64\_ADR\_PREL\_PG\_HI21 against external symbol `\_\_tracepoint\_cfg80211\_rx\_unexpected\_4addr\_frame' can not be used when making a shared object; recompile with -fPIC

realme-c21-kernel/kernel/out/../include/linux/compiler.h:193: undefined reference to `\_\_tracepoint\_cfg80211\_rx\_unexpected\_4addr\_frame'

realme-c21-kernel/kernel/out/../include/linux/compiler.h:193: undefined reference to `\_\_tracepoint\_cfg80211\_return\_bool'

realme-c21-kernel/gcc-4.9/bin/aarch64-linux-android-ld: net/wireless/nl80211.o: relocation R\_AARCH64\_ADR\_PREL\_PG\_HI21 against external symbol `\_\_tracepoint\_cfg80211\_return\_bool' can not be used when making a shared object; recompile with -fPIC

realme-c21-kernel/kernel/out/../include/linux/compiler.h:193: undefined reference to `\_\_tracepoint\_cfg80211\_return\_bool'

realme-c21-kernel/kernel/out/../include/linux/compiler.h:193: undefined reference to `\_\_tracepoint\_cfg80211\_return\_bool'

realme-c21-kernel/gcc-4.9/bin/aarch64-linux-android-ld: net/wireless/nl80211.o: relocation R\_AARCH64\_ADR\_PREL\_PG\_HI21 against external symbol `\_\_tracepoint\_cfg80211\_return\_bool' can not be used when making a shared object; recompile with -fPIC

realme-c21-kernel/kernel/out/../include/linux/compiler.h:193: undefined reference to `\_\_tracepoint\_cfg80211\_return\_bool'

realme-c21-kernel/kernel/out/../include/linux/compiler.h:193: undefined reference to `\_\_tracepoint\_cfg80211\_return\_bool'

realme-c21-kernel/gcc-4.9/bin/aarch64-linux-android-ld: net/wireless/nl80211.o: relocation R\_AARCH64\_ADR\_PREL\_PG\_HI21 against external symbol `\_\_tracepoint\_cfg80211\_return\_bool' can not be used when making a shared object; recompile with -fPIC

net/wireless/nl80211.o:realme-c21-kernel/kernel/out/../include/linux/compiler.h:193: more undefined references to `\_\_tracepoint\_cfg80211\_return\_bool' follow

realme-c21-kernel/gcc-4.9/bin/aarch64-linux-android-ld: net/wireless/nl80211.o: relocation R\_AARCH64\_ADR\_PREL\_PG\_HI21 against external symbol `\_\_tracepoint\_cfg80211\_return\_bool' can not be used when making a shared object; recompile with -fPIC

net/wireless/nl80211.o: In function `\_\_read\_once\_size':
````




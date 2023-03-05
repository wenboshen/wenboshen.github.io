---
layout: post
title: "Compile (GCC and Clang) and QEMU Linux Kernel"
date: 2017-07-01
description: ""
category: 
tags: []
---
* TOC
{:toc}

### Setup Buildroot
1. Download the source 
```
git clone git://git.buildroot.net/buildroot buildroot.git
cd buildroot.git
```

2. Configure the buildroot for aarch64 and `make menuconfig; make`  

For X86-64, change the following options, 
Target Options -> Target Architecture(X86_64)  
Filesystem images -> cpio the root filesystem (for use as an initial RAM filesystem)   

For AArch64, similar to X86  
Target Options -> Target Architecture(AArch64 (little endian))  
Toolchain -> Toolchain type (External toolchain)  
Toolchain -> Toolchain (Linaro AArch64 2016.11)  
System configuration -> Run a getty after boot -> TTY port console  
Filesystem images -> cpio the root filesystem (for use as an initial RAM filesystem)   

The output cpio will be in output/images/rootfs.cpio


### Setup QEMU
For X86/X86_64, `sudo apt-get install qemu-system-x86`  
For AArch64, need to build qemu-system-aarch64 locally.

1. Download qemu git, check out the branch and check out the latest branch
```
git clone git://git.qemu.org/qemu.git qemu.git
git branch -a
git checkout -b stable-2.6 origin/stable-2.6
```

2. Install the build dependency and compile qemu for aarch64
```
sudo apt-get build-dep qemu
```

3. Start compile
```
./configure --target-list=aarch64-softmmu
make -j4
```
The compilation should be done in several minutes. The generated file should be
aarch64-softmmu/qemu-system-aarch64

### Compile Linux Kernel
#### Using GCC
```
# For X86_64
make defconfig
make -j16 2>&1 | tee build.log

# For AArch64
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 2>&1 | tee build.log
```

#### Using LLVM/Clang
```
# For X86_64
make CC=clang defconfig
make CC=clang -j16 2>&1 | tee build.log

# For AArch64
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- HOSTCC=clang CC=clang defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
	HOSTCC=clang CC=clang -j16 2>&1 | tee build.log
```

### Run Kernel on QEMU
```
# For X86_64
qemu-system-x86_64 \
-nographic -smp 8 -m 2048 \
-kernel $KERNEL_DIR/arch/x86/boot/bzImage \
-initrd $ROOTFS_DIR/output/images/rootfs.cpio \
-append "console=ttyS0" 

# For AArch64
$QEMU_DIR/aarch64-softmmu/qemu-system-aarch64 \
-machine virt -cpu cortex-a57 \
-nographic -smp 8 -m 2048 \
-kernel $KERNEL_DIR/arch/arm64/boot/Image \
-initrd $ROOTFS_DIR/output/images/rootfs.cpio \
-append "console=ttyAMA0" 
```

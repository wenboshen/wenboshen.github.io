---
title: "Kernel Forward Control Flow Integrity based on Clang"
collection: posts
permalink: /posts/2018-08-17-kernel-cfi
date: 2018-08-17
description: ""
category: 
tags: []
---
* TOC
{:toc}

### Get build tools and source code

```
git clone https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/ clang
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9 gcc

git clone https://android.googlesource.com/kernel/common
git checkout android-4.14
```
Add source code in [hello-cfi](https://github.com/wenboshen/hello-cfi), modify
`kernel/drivers/misc/Makefile` to including these two file. 


### Config, compile, and run Linux kernel

```
# In file arch/arm64/configs/defconfig

CONFIG_ARM64_VA_BITS_39=y
CONFIG_GDB_SCRIPTS=y 
CONFIG_KVM=n

# Enable Clang CFI
CONFIG_LTO_CLANG=y
CONFIG_CFI_CLANG=y


# Compile
DIR=/home/wenbo/KERNEL/CFI_TOOLCHAIN
export PATH=$DIR/gcc/bin:$PATH

//git checkout 8b733c1257bb0229c8a32974c8c615d7dc1f3d18 if clang-4679922 is missing
export PATH=$DIR/clang/clang-4679922/bin:$PATH
export LD_LIBRARY_PATH=$DIR/clang/clang-4679922/lib64:$PATH

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-android- CLANG_TRIPLE=aarch64-linux-gnu- CC=clang defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-android- CLANG_TRIPLE=aarch64-linux-gnu- HOSTCC=clang CC=clang -j4 2>&1 | tee build.log

# Run in QEMU
$QEMU_DIR/aarch64-softmmu/qemu-system-aarch64 \
-machine virt -cpu cortex-a57 -machine type=virt \
-nographic -smp 8 -m 2048 \
-kernel $KERNEL_DIR/arch/arm64/boot/Image \
-initrd ../rootfs.cpio \
--append "console=ttyAMA0"  -s

# Trigger CFI
cat /proc/hello_cfi
echo 0 > /proc/hello_cfi #jump to int_arg
echo 1 > /proc/hello_cfi #jump to bad_int_arg
echo 2 > /proc/hello_cfi #jump to float_arg
echo 3 > /proc/hello_cfi #jump to not_entry_point begin
echo 4 > /proc/hello_cfi #jump to not_entry_point middle
```

### How Kernel CFI works
Without Kernel CFI, a function pointer can jump to anywhere.
Therefore, all of the above 5 cases work.

For Kernel CFI, same with the [usespace CFI](https://wenboshen.org/posts/2017-03-20-clang-cfi.html), Clang CFI will only a function pointer jump to targets with the same argument signature. 

It will classify functions into different sets according to their sigatures. In this way,  `int_arg` and `bad_int_arg` will be in the same set, along with other function with `int arg` argument. And then it uses a bit map to filter the jump target at run time and panics if the jump target is out of the set.  The assembly code is
```assembly
ffffff8008b07848:	f00040a9 	adrp	x9, ffffff800931e000 <nfs4_inode_return_delegation>
ffffff8008b0784c:	911d2129 	add	x9, x9, #0x748 //load the first func addr of this set
ffffff8008b07850:	d100c115 	sub	x21, x8, #0x30
ffffff8008b07854:	b000a2e8 	adrp	x8, ffffff8009f64000 <blkfront_mutex+0x28>
ffffff8008b07858:	910aa108 	add	x8, x8, #0x2a8
ffffff8008b0785c:	f8757908 	ldr	x8, [x8,x21,lsl #3]
ffffff8008b07860:	cb090109 	sub	x9, x8, x9
ffffff8008b07864:	93c90929 	ror	x9, x9, #2
ffffff8008b07868:	f1012d3f 	cmp	x9, #0x4b //this set only contains 0x4b functions
ffffff8008b0786c:	54000202 	b.cs	ffffff8008b078ac <hello_write.cfi+0x10c> //flow path is not in set
ffffff8008b07870:	2a1503e0 	mov	w0, w21
ffffff8008b07874:	d63f0100 	blr	x8
......
ffffff8008b078ac:	d2874840 	mov	x0, #0x3a42                	// #14914
ffffff8008b078b0:	f2b0a680 	movk	x0, #0x8534, lsl #16
ffffff8008b078b4:	f2c02b40 	movk	x0, #0x15a, lsl #32
ffffff8008b078b8:	f2e8f9c0 	movk	x0, #0x47ce, lsl #48
ffffff8008b078bc:	aa0803e1 	mov	x1, x8
ffffff8008b078c0:	aa1f03e2 	mov	x2, xzr
ffffff8008b078c4:	f90007e8 	str	x8, [sp,#8]
ffffff8008b078c8:	97e0d1f3 	bl	ffffff800833c094 <__cfi_slowpath>
ffffff8008b078cc:	f94007e8 	ldr	x8, [sp,#8]
ffffff8008b078d0:	17ffffe8 	b	ffffff8008b07870 <hello_write.cfi+0xd0>
```

The result is
```
echo 0 > /proc/hello_cfi #jump to int_arg		--> Good
echo 1 > /proc/hello_cfi #jump to bad_int_arg		--> Good
echo 2 > /proc/hello_cfi #jump to float_arg		--> Panic
echo 3 > /proc/hello_cfi #jump to not_entry_point begin	--> Panic
echo 4 > /proc/hello_cfi #jump to not_entry_point middle--> Panic
```

[Control Flow Integrity in the Android kernel](https://android-developers.googleblog.com/2018/10/control-flow-integrity-in-android-kernel.html) shows that 55% indirected function calls have less than 5 jump targets, but still 45% have more than 5 targets.

### `vfs_write` example
In [android common kernel](https://android.googlesource.com/kernel/common), branch android-4.14, commit 70014b13c28cbf1e,
`__vfs_write` calls `call_write_iter`, which is
```
static inline ssize_t call_write_iter(struct file *file, struct kiocb *kio, struct iov_iter *iter)
{
	return file->f_op->write_iter(kio, iter);
}

// The indirect callsite is compiled into the following assembly
ffffff8008472cb4:	b00072e9 	adrp	x9, ffffff80092cf000 <bcm2835_clk_probe>
ffffff8008472cb8:	911e2129 	add	x9, x9, #0x788
ffffff8008472cbc:	cb090109 	sub	x9, x8, x9
ffffff8008472cc0:	93c90929 	ror	x9, x9, #2
ffffff8008472cc4:	1a9f054a 	csinc	w10, w10, wzr, eq
ffffff8008472cc8:	f100993f 	cmp	x9, #0x26
ffffff8008472ccc:	320003e9 	orr	w9, wzr, #0x1
ffffff8008472cd0:	b90013ea 	str	w10, [sp,#16]
ffffff8008472cd4:	a902a7eb 	stp	x11, x9, [sp,#40]
ffffff8008472cd8:	54000322 	b.cs	ffffff8008472d3c <__vfs_write.cfi+0x188>
ffffff8008472cdc:	9100e3e0 	add	x0, sp, #0x38
ffffff8008472ce0:	910043e1 	add	x1, sp, #0x10
ffffff8008472ce4:	d63f0100 	blr	x8
```
The jump targets set contains 0x26 functions, as shown below, in which the read, direct_IO are also included.
```
ffffff80092cf788 t devkmsg_write
ffffff80092cf78c T generic_write_checks
ffffff80092cf790 T generic_file_write_iter
ffffff80092cf794 T __generic_file_write_iter
ffffff80092cf798 T generic_file_direct_write
ffffff80092cf79c T generic_file_read_iter
ffffff80092cf7a0 t pipe_read
ffffff80092cf7a4 t pipe_write
ffffff80092cf7a8 t blkdev_direct_IO
ffffff80092cf7ac T blkdev_write_iter
ffffff80092cf7b0 T blkdev_read_iter
ffffff80092cf7b4 t ext4_file_read_iter
ffffff80092cf7b8 t ext4_file_write_iter
ffffff80092cf7bc t ext4_direct_IO
ffffff80092cf7c0 t ext2_file_read_iter
ffffff80092cf7c4 t ext2_file_write_iter
ffffff80092cf7c8 t ext2_direct_IO
ffffff80092cf7cc t hugetlbfs_read_iter
ffffff80092cf7d0 t fat_direct_IO
ffffff80092cf7d4 T nfs_file_write
ffffff80092cf7d8 T nfs_file_read
ffffff80092cf7dc T nfs_direct_IO
ffffff80092cf7e0 T nfs_file_direct_read
ffffff80092cf7e4 T nfs_file_direct_write
ffffff80092cf7e8 t v9fs_direct_IO
ffffff80092cf7ec t v9fs_file_read_iter
ffffff80092cf7f0 t v9fs_file_write_iter
ffffff80092cf7f4 t v9fs_mmap_file_read_iter
ffffff80092cf7f8 t v9fs_mmap_file_write_iter
ffffff80092cf7fc t read_iter_zero
ffffff80092cf800 t write_iter_null
ffffff80092cf804 t read_iter_null
ffffff80092cf808 t tun_chr_read_iter
ffffff80092cf80c t tun_chr_write_iter
ffffff80092cf810 t snd_pcm_writev
ffffff80092cf814 t snd_pcm_readv
ffffff80092cf818 t sock_read_iter
ffffff80092cf81c t sock_write_iter
```


### Clang 3.6 CFI implementation
In LLVM/Clang 3.6, CFI Indirect Call is implemented in two files [./lib/CodeGen/ForwardControlFlowIntegrity.cpp](https://github.com/llvm-mirror/llvm/blob/release_36/lib/CodeGen/ForwardControlFlowIntegrity.cpp) and [./include/llvm/CodeGen/ForwardControlFlowIntegrity.h](https://github.com/llvm-mirror/llvm/blob/release_36/include/llvm/CodeGen/ForwardControlFlowIntegrity.h).

The whole process contains roughly 2 steps. First, generate the function pointer to all jump targets mapping; Second, rewrite the jump instruction.
The first step is done in [`runOnModule`](https://github.com/llvm-mirror/llvm/blob/release_36/lib/CodeGen/ForwardControlFlowIntegrity.cpp#L175) via
[`JumpInstrTableInfo`](https://github.com/llvm-mirror/llvm/blob/release_36/include/llvm/Analysis/JumpInstrTableInfo.h#L35).
```
class JumpInstrTableInfo : public ImmutablePass {
public:
  ......
  typedef std::pair<Function *, Function *> JumpPair;
  typedef DenseMap<FunctionType *, std::vector<JumpPair> > JumpTables;

  /// Inserts an entry in a table, adding the table if it doesn't exist.
  void insertEntry(FunctionType *TableFunTy, Function *Target, Function *Jump);

  /// Gets the tables.
  const JumpTables &getTables() const { return Tables; }
  JumpTables Tables;
  ......
};

void JumpInstrTableInfo::insertEntry(FunctionType *TableFunTy, Function *Target,
                                     Function *Jump) {
  Tables[TableFunTy].push_back(JumpPair(Target, Jump));
}
```
From the above code, we know that JumpInstrTableInfo organizes PointerType, Target and Jump in the Tables.
With all these info, the second step is easy, only needs to get the mask values from the size and rewrite the jump instruction.
The critical data structures are listed below.
```
typedef SmallVector<Instruction *, 64> CallSet;

/// A structure that is used to keep track of constant table information.
struct CFIConstants {
  Constant *StartValue;
  Constant *MaskValue;
  Constant *Size;
};

/// A map from function type to the base of the table for this type and a mask
/// for the table
typedef DenseMap<FunctionType *, CFIConstants> CFITables;

CallSet IndirectCalls;

/// The type of jumptable implementation.
JumpTable::JumpTableType JTType;

/// The type of CFI check to add before each indirect call.
CFIntegrity CFIType;

namespace JumpTable {
  enum JumpTableType {
    Single,          // Use a single table for all indirect jumptable calls.
    Arity,           // Use one table per number of function parameters.
    Simplified,      // Use one table per function type, with types projected
                     // into 4 types: pointer to non-function, struct,
                     // primitive, and function pointer.
    Full             // Use one table per unique function type
  };
}
```

### Clang 6.0 CFI implementation
In LLVM/Clang 6.0, CFI code changes a lot. The main code is in [lib/Transforms/IPO/LowerTypeTests.cpp](https://github.com/llvm-mirror/llvm/blob/master/lib/Transforms/IPO/LowerTypeTests.cpp), also some code in clang/lib/CodeGen (grep for CFIICall) [[2]](https://lists.llvm.org/pipermail/llvm-dev/2018-August/125159.html).

The jump table is generated in [`buildBitSetsFromFunctionsNative`](https://github.com/llvm-mirror/llvm/blob/release_60/lib/Transforms/IPO/LowerTypeTests.cpp#L1270).


### References
1. [Kernel Control Flow Integrity](https://source.android.com/devices/tech/debug/kcfi)
2. [Control Flow Integrity in the Android kernel](https://android-developers.googleblog.com/2018/10/control-flow-integrity-in-android-kernel.html)
3. [Which are CFI (Control Flow Integrity) related files](https://lists.llvm.org/pipermail/llvm-dev/2018-August/125159.html)

| 项目              | GNU                       | LLVM          |
| --------------- | ------------------------- | ------------- |
| C 编译器           | `gcc`                     | `clang`       |
| C++编译器          | `g++`                     | `clang++`     |
| binutils        | GNU binutils              | LLVM Core     |
| 链接器(linker)     | `ld`, `ld.bfd`, `ld.gold` | `lld`         |
| 运行时(intrinsics) | `libgcc`                  | `compiler-rt` |
| 原子操作            | `libatomic`               | `compiler-rt` |
| 栈展开(unwind)     | `libgcc_s`                | `libunwind`   |
| C 语言库           | `glibc`                   | `libc`        |
| C++ 标准库         | `libstdc++`               | `libcxx`      |
| C++ ABI         | `Libsupcxx`               | `libcxxabi`   |
| 汇编器             | `as`                      | `llvm-as`     |
| 调试器             | `gdb`                     | `lldb`        |

#https://github.com/loongson/la-asm-manual  
#https://github.com/loongson/LoongArch-Documentation  
#rustc的linker-plugin-lto功能需要clang和依赖库需要启用lto重新编译  
#llvm clang lld(mold) 原生支持多架构,交叉编译简单  

# ubuntu_x86_64下 交叉编译 loongarch64

### 0.下载并解压文件

```bash
$ mkdir -p ~/.loongarch && cd ~/.loongarch

$ wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/CLFS-loongarch64-8.1-x86_64-cross-tools-gcc-glibc.tar.xz && tar -xvJf CLFS-loongarch64-8.1-x86_64-cross-tools-gcc-glibc.tar.xz
$ wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/qemu-loongarch64 && chmod +x qemu-loongarch64
$ # 可选下载,运行时可能缺少动态库
$ wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/clfs-loongarch64-system-8.1-sysroot.squashfs && unsquashfs -user-xattrs clfs-system-8.1-sysroot.loongarch64.squashfs 
```

```
$ tree -L 2 .loongarch/
.loongarch/
├── cross-tools
│   ├── bin
│   ├── include
│   ├── lib
│   ├── libexec
│   ├── loongarch64-unknown-linux-gnu
│   ├── share
│   └── target
├── qemu-loongarch64
└── squashfs-root
    ├── bin -> usr/bin
    ├── boot
    ├── dev
    ├── etc
    ├── home
    ├── lib -> usr/lib
    ├── lib64 -> usr/lib64
    ├── media
    ├── mnt
    ├── opt
    ├── proc
    ├── root
    ├── run
    ├── sbin -> usr/sbin
    ├── srv
    ├── sys
    ├── tmp
    ├── usr
    └── var
```

### 1. rust 1.72 添加 target loongarch64-unknown-linux-gnu

```bash
$ rustup self update
$ rustup update stable
$ rustup target add loongarch64-unknown-linux-gnu
```

### 2. 修改 ~/.cargo/config.toml

```
[target.loongarch64-unknown-linux-gnu]
linker = "loongarch64-unknown-linux-gnu-gcc"
runner = "qemu-loongarch64"
```

### 3.修改 ~/.bashrc

```
loongarch.env(){
    export LOONGARCH_HOME=$HOME/.loongarch
    export LOONGARCH_SYSTEM_SYSROOT=$LOONGARCH_HOME/squashfs-root   

    export LOONGARCH_TOOLS_DIR=$LOONGARCH_HOME/cross-tools   
    export PATH=$LOONGARCH_HOME:$PATH        
    export PATH=$LOONGARCH_TOOLS_DIR/bin:$PATH    
    export LD_LIBRARY_PATH=$LOONGARCH_TOOLS_DIR/lib:$LOONGARCH_TOOLS_DIR/loongarch64-unknown-linux-gnu/lib:$LD_LIBRARY_PATH

    if [[ -e "$LOONGARCH_SYSTEM_SYSROOT/usr"  && -n $LOONGARCH_SYSTEM_SYSROOT ]]
    then
        export QEMU_LD_PREFIX=$LOONGARCH_SYSTEM_SYSROOT
    else
        export QEMU_LD_PREFIX=$LOONGARCH_TOOLS_DIR/target
    fi

    export DISPLAY=:0

    echo ""
    echo "##############info##################"
    which loongarch64-unknown-linux-gnu-gcc
    which qemu-loongarch64
    echo LOONGARCH_SYSTEM_SYSROOT=$LOONGARCH_SYSTEM_SYSROOT
    echo QEMU_LD_PREFIX=$QEMU_LD_PREFIX
    echo LD_LIBRARY_PATH=$LD_LIBRARY_PATH
    echo LOONGARCH_HOME=$LOONGARCH_HOME
    echo "####################################"
    echo ""   
}

loongarch.cargo(){
    if [ -z `which loongarch64-unknown-linux-gnu-gcc` ] 
    then
        loongarch.env
    fi

    cargo update

    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_LINKER="loongarch64-unknown-linux-gnu-gcc" \
    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_RUNNER="qemu-loongarch64" \
    CARGO_BUILD_TARGET="loongarch64-unknown-linux-gnu" cargo $@
}
```

### 4.测试

```bash
$ source ~/.bashrc                
$ loongarch.env
$ cargo new hello && cd hello
$ cargo run --target loongarch64-unknown-linux-gnu
```

```bash
$ source ~/.bashrc                
$ loongarch.env
$ git clone --depth=1 https://github.com/extrawurst/gitui
$ cd gitui
$ cargo update
$ cargo run --release --target loongarch64-unknown-linux-gnu
```

---

# 使用x86_64(llvm clang lld  rust1.72 qemu-loongarch64) 和 clfs-loongarch64-system-8.1-sysroot.squashfs 交叉编译loongarch64

### 0. 编译llvm core, clang, lld

> ```bash
> $ mkdir -p ~/.loongarch/llvm
> 
> $ mkdir -p ~/.loongarch/llvm/build/llvm
> $ mkdir -p ~/.loongarch/llvm/install/llvm
> 
> $ mkdir -p ~/.loongarch/llvm/build/clang
> $ mkdir -p ~/.loongarch/llvm/install/clang
> 
> $ mkdir -p ~/.loongarch/llvm/build/lld
> $ mkdir -p ~/.loongarch/llvm/install/lld
> 
> $ cd ~/.loongarch/llvm
> 
> $ git clone --depth 1 https://github.com/llvm/llvm-project.git
> 
> $ #build llvm
> $ cmake -G Ninja -S ~/.loongarch/llvm/llvm-project/llvm -B ~/.loongarch/llvm/build/llvm -DLLVM_INSTALL_UTILS=ON -DCMAKE_INSTALL_PREFIX=~/.loongarch/llvm/install/llvm -DCMAKE_BUILD_TYPE=Release
> $ ninja -C ~/.loongarch/llvm/build/llvm install
> 
> $ #build clang
> $ cmake -G Ninja -S ~/.loongarch/llvm/llvm-project/clang -B ~/.loongarch/llvm/build/clang -DLLVM_EXTERNAL_LIT=~/.loongarch/llvm/build/llvm/utils/lit -DLLVM_ROOT=~/.loongarch/llvm/install/llvm -DCMAKE_INSTALL_PREFIX=~/.loongarch/llvm/install/clang -DCMAKE_BUILD_TYPE=Release
> $ ninja -C ~/.llvm_dev/build/clang install
> 
> $ #build lld
> $ cmake -G Ninja -S ~/.loongarch/llvm/llvm-project/lld -B ~/.loongarch/llvm/build/lld -DLLVM_EXTERNAL_LIT=~/.loongarch/llvm/build/llvm/utils/lit -DLLVM_ROOT=~/.loongarch/llvm/install/llvm -DCMAKE_INSTALL_PREFIX=~/.loongarch/llvm/install/lld -DCMAKE_BUILD_TYPE=Release
> $ ninja -C ~/.loongarch/llvm/build/lld install
> 
> $ cd ~/.loongarch && ln -s llvm/install llvm-18git
> ```

### 1.下载qemu-loonarch64 和 sysroot

```bash
$ mkdir -p ~/.loongarch && cd ~/.loongarch
$ wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/qemu-loongarch64 && chmod +x qemu-loongarch64
$ wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/clfs-loongarch64-system-8.1-sysroot.squashfs && unsquashfs -user-xattrs clfs-system-8.1-sysroot.loongarch64.squashfs 
```

#多架构的小sysroot
#https://github.com/crosstool-ng/crosstool-ng  
#https://github.com/sunfishcode/eyra  
#https://github.com/sunfishcode/origin-studio  
#编译loongarch64(compiler-rt,libc,libunwind,libcxx,libcxxabi)   
#zig  
#多架构clfs + 统一app安装包格式 + 跨系统和架构的(共享库管理工具+包管理工具+app商店) + 各系统默认打包格式(补充)

### 2. rust 1.72 添加 target loongarch64-unknown-linux-gnu

```bash
$ rustup self update
$ rustup update stable
$ rustup target add loongarch64-unknown-linux-gnu
```

### 3. 修改~/.bashrc

```bash
loongarch.cargo-sysroot(){
    export LOONGARCH_HOME=~/.loongarch
    export LLVM_TOOLS_HOME=$LOONGARCH_HOME/llvm-18git
    export LOONGARCH_SYSROOT=$LOONGARCH_HOME/squashfs-root
    export PATH=$LLVM_TOOLS_HOME/clang/bin:$LLVM_TOOLS_HOME/lld/bin:$LLVM_TOOLS_HOME/llvm/bin:$LOONGARCH_HOME:$PATH
    #debug_rustflags="-C opt-level=0 -C strip=none -C split-debuginfo=packed -C symbol-mangling-version=v0 -C debuginfo=2 -C debug-assertions=true"

    CC_loongarch64_unknown_linux_gnu="clang" \
    CFLAGS_loongarch64_unknown_linux_gnu="--sysroot=$LOONGARCH_SYSROOT" \
    AR_loongarch64_unknown_linux_gnu="llvm-ar" \
    RANLIB_loongarch64_unknown_linux_gnu="llvm-ranlib" \
    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_LINKER="clang" \
    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_AR="llvm-ar" \
    QEMU_LD_PREFIX=$LOONGARCH_SYSROOT \
    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_RUNNER="$LOONGARCH_HOME/qemu-loongarch64" \
    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_RUSTFLAGS="-C link-arg=-fuse-ld=lld -C link-arg=--target=loongarch64-unknown-linux-gnu -C link-args=--sysroot=$LOONGARCH_SYSROOT -C target-feature=+crt-static" \
    CARGO_BUILD_TARGET="loongarch64-unknown-linux-gnu" cargo  $@

}


loongarch.cargo-optimized-sysroot(){
    export LOONGARCH_HOME=~/.loongarch
    export LOONGARCH_SYSROOT=$LOONGARCH_HOME/squashfs-root
    export QEMU_LD_PREFIX=$LOONGARCH_SYSROOT
    export PATH=$LOONGARCH_HOME/llvm-18git/clang/bin:$LOONGARCH_HOME/llvm-18git/lld/bin:$LOONGARCH_HOME/llvm-18git/llvm/bin:$LOONGARCH_HOME:$PATH

    runner="qemu-loongarch64"
    cflags="--sysroot=$LOONGARCH_SYSROOT"

    #性能优化参考: https://github.com/nnethercote/perf-book/blob/master/src/SUMMARY.md
    #profdata文件生成: 参考https://doc.rust-lang.org/rustc/profile-guided-optimization.html
    #profdata=
    #profile_guided_optimizion="-C llvm-args=-pgo-warn-missing-function -C profile-use=$profdata"

    #参考https://doc.rust-lang.org/cargo/reference/profiles.html#panic
    #在panic时,不需要unwind,如不使用[catch_unwind](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html),可以启用,提示panic=abort出错时注释该行
    panic="-C panic=abort"
    #参考https://doc.rust-lang.org/cargo/reference/profiles.html#opt-level
    optimized_level="-C opt-level=3"
    #参考https://doc.rust-lang.org/rustc/linker-plugin-lto.html
    #optimizied_linker_plugin_lto="-C linker-plugin-lto -C linker=clang"
    #参考https://doc.rust-lang.org/cargo/reference/profiles.html#lto
    optimizied_lto="-C lto=fat -C embed-bitcode=yes $optimizied_linker_plugin_lto"
    #参考https://doc.rust-lang.org/rustc/codegen-options/index.html#codegen-units
    optimized_speed="-C codegen-units=1 $panic $optimizied_lto $optimized_level"

    #参考https://doc.rust-lang.org/rustc/codegen-options/index.html#strip
    #https://github.com/johnthagen/min-sized-rust
    #optimized_size="-C strip=symbols"

    optimized_linking_times="-C link-arg=-fuse-ld=lld -C link-arg=--target=loongarch64-unknown-linux-gnu -C link-arg=--sysroot=$LOONGARCH_SYSROOT"

    #参考https://doc.rust-lang.org/rustc/codegen-options/index.html#target-feature
    #https://doc.rust-lang.org/rustc/codegen-options/index.html#relocation-model
    #rustc --print target-features --target loongarch64-unknown-linux-gnu
    #rustc --print target-cpus --target loongarch64-unknown-linux-gnu
    #rustc --print relocation-models --target loongarch64-unknown-linux-gnu 
    target_feature="-C target-feature=+crt-static"

    rustflags="$optimized_linking_times $optimized_speed $optimized_size $target_feature $profile_guided_optimizion"

    CC_loongarch64_unknown_linux_gnu="clang" \
    CFLAGS_loongarch64_unknown_linux_gnu="$cflags" \
    AR_loongarch64_unknown_linux_gnu="llvm-ar" \
    RANLIB_loongarch64_unknown_linux_gnu="llvm-ranlib" \
    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_LINKER="clang" \
    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_AR="llvm-ar" \
    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_RUNNER="$runner" \
    CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_RUSTFLAGS="$rustflags" \
    CARGO_BUILD_TARGET="loongarch64-unknown-linux-gnu" cargo  $@     
}

loongarch.clang(){
    export LOONGARCH_HOME=~/.loongarch
    export LOONGARCH_SYSROOT=$LOONGARCH_HOME/squashfs-root
    export LLVM_TOOLS_HOME=$LOONGARCH_HOME/llvm-18git
    export PATH=$LLVM_TOOLS_HOME/clang/bin:$LLVM_TOOLS_HOME/lld/bin:$LLVM_TOOLS_HOME/llvm/bin:$LOONGARCH_HOME:$PATH
    clang  --target=loongarch64-unknown-linux-gnu  -fuse-ld=lld --sysroot=$LOONGARCH_SYSROOT   $@
}
```

### 4.测试

```bash
$ source ~/.bashrc                
$ cargo new hello && cd hello
$ loongarch.cargo-sysroot run 
```

```bash
$ source ~/.bashrc                
$ git clone --depth=1 https://github.com/extrawurst/gitui
$ cd gitui
$ cargo update
$ loongarch.cargo-sysroot build

$ file target/loongarch64-unknown-linux-gnu/debug/gitui
target/loongarch64-unknown-linux-gnu/debug/gitui: ELF 64-bit LSB executable, LoongArch, version 1 (SYSV), statically linked, for GNU/Linux 5.19.0, with debug_info, not stripped
```

```bash
$ source ~/.bashrc
$ git clone --depth=1 https://github.com/uutils/coreutils
$ cd coreutils
$ cargo update
$ loongarch.cargo-optimized-sysroot build --release --features unix

$ file target/loongarch64-unknown-linux-gnu/release/coreutils
target/loongarch64-unknown-linux-gnu/release/coreutils: ELF 64-bit LSB executable, LoongArch, version 1 (SYSV), statically linked, for GNU/Linux 5.19.0, stripped

$ du -h target/loongarch64-unknown-linux-gnu/release/coreutils
13M    target/loongarch64-unknown-linux-gnu/release/coreutils

$ qemu-loongarch64 target/loongarch64-unknown-linux-gnu/release/coreutils -h
coreutils 0.0.21 (multi-call binary)

Usage: coreutils [function [arguments...]]

Currently defined functions:

    [, arch, b2sum, b3sum, base32, base64, basename, basenc, cat, chgrp, chmod, chown, chroot,
    cksum, comm, cp, csplit, cut, date, dd, df, dir, dircolors, dirname, du, echo, env, expand,
    expr, factor, false, fmt, fold, groups, hashsum, head, hostid, hostname, id, install, join,
    kill, link, ln, logname, ls, md5sum, mkdir, mkfifo, mknod, mktemp, more, mv, nice, nl,
    nohup, nproc, numfmt, od, paste, pathchk, pinky, pr, printenv, printf, ptx, pwd, readlink,
    realpath, relpath, rm, rmdir, seq, sha1sum, sha224sum, sha256sum, sha3-224sum, sha3-256sum,
    sha3-384sum, sha3-512sum, sha384sum, sha3sum, sha512sum, shake128sum, shake256sum, shred,
    shuf, sleep, sort, split, stat, stdbuf, stty, sum, sync, tac, tail, tee, test, timeout,
    touch, tr, true, truncate, tsort, tty, uname, unexpand, uniq, unlink, uptime, users, vdir,
    wc, who, whoami, yes
```

---

# x86_64(qemu-loongarch64 chroot) 和 clfs-loongarch64-system-8.1-sysroot.squashfs

### 0.下载qemu-loonarch64 和 sysroot

```bash
$ mkdir -p ~/.loongarch && cd ~/.loongarch
$ wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/qemu-loongarch64 && chmod +x qemu-loongarch64
$ wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/clfs-loongarch64-system-8.1-sysroot.squashfs && unsquashfs -user-xattrs clfs-system-8.1-sysroot.loongarch64.squashfs 
```

### 1.chroot 环境准备

```bash
$ #存在register status
$ ls /proc/sys/fs/binfmt_misc
register status

$ #status 为enabled
$ cat /proc/sys/fs/binfmt_misc/status
enabled

$ #是否存在qemu-loongarch64规则,不存在则使用check_binfmt_misc_for_qemu_loongarch64生成创建命令并执行
$ ls /proc/sys/fs/binfmt_misc
register status qemu-loongarch64


$ #1为启用,0为禁用,-1为删除qemu-loongarch64规则
$ #sudo bash -c 'echo -1 > /proc/sys/fs/binfmt_misc/qemu-loongarch64'

$ #复制qemu-loongarch64到sysroot,确保qemu-loongarch64在PATH=~/.loongarch:$PATH
$ cp --parents `which qemu-loongarch64` ~/.loongarch/squashfs-root

#确认interpreter路径与~/.loongarch/squashfs-root下的一致
$ cat /proc/sys/fs/binfmt_misc/qemu-loongarch64
```

```bash
check_binfmt_misc_for_qemu_loongarch64(){
    PATH=~/.loongarch:$PATH
    qemu_file_name="qemu-loongarch64"
    if [ `which $qemu_file_name` ];then    
        qemu_match=":${qemu_file_name}:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x02\x01:\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:`which $qemu_file_name`:"
        echo "使用以下命令生成的binfmt_misc匹配规则"
        echo -n "sudo bash -c "
        echo -n "'" 
        echo -n "echo " 
        echo -n '"'
        echo -n ${qemu_match}
        echo -n '"'
        echo -n ' > '
        echo -n '/proc/sys/fs/binfmt_misc/register'
        echo "'"
    else
        echo "$qemu_file_name not found"
    fi
}
```

### 2. chroot

```bash
$ sudo chroot ~/.loongarch/squashfs-root
$ mount -t proc proc /proc
$ mount -t sysfs sys /sys
$ mount -t devtmpfs dev /dev
$ mount -t devpts devpts /dev/pts
$ mount -t tmpfs shmfs /dev/shm
$ useradd -m larch
$ su larch
$ cd ~ && uname -m
loongarch64
$ exit
$ exit
$ uname -m
x86_64
```

---

# emulator-loongarch64
#virt-manager
#tcg
#kvm
#qemu
#StratoVirt
#crosvm

 
```bash
$ #编译 x86_64(qemu-system-loongarch64) 参考https://wiki.qemu.org/Hosts/Linux
$ mkdir -p ~/.loongarch && cd ~/.loongarch
$ mkdir -p ~/.loongarch/emulator-loongarch64
$ cd ~/.loongarch/emulator-loongarch64
$ git clone git://git.qemu-project.org/qemu.git
$ mkdir -p ~/.loongarch/emulator-loongarch64/qemu-system-build
$ mkdir -p ~/.loongarch/emulator-loongarch64/qemu-system
$ cd ~/.loongarch/emulator-loongarch64/qemu-system-build
$ #按提示安装缺少的依赖库
$ ../qemu/configure --target-list="loongarch64-softmmu loongarch64-linux-user" --enable-lto --prefix=`pwd`/../qemu-system
$ make -j16
$ make install
$ #文件在~/.loongarch/emulator-loongarch64/qemu-system/bin/qemu-system-loongarch64
$ ~/.loongarch/emulator-loongarch64/qemu-system/bin/qemu-system-loongarch64 -M ?
Supported machines are:
none                 empty machine
virt                 Loongson-3A5000 LS7A1000 machine (default)

$ ~/.loongarch/emulator-loongarch64/qemu-system/bin/qemu-system-loongarch64 -cpu ?
la132-loongarch-cpu
la464-loongarch-cpu

$ ~/.loongarch/emulator-loongarch64/qemu-system/bin/qemu-system-loongarch64  -machine virt,help
virt-machine options:
  acpi=<OnOffAuto>       - Enable ACPI
  append=<string>        - Linux kernel command line
  boot=<BootConfiguration> - Boot configuration
  confidential-guest-support=<link<confidential-guest-support>> - Set confidential guest scheme to support
  dt-compatible=<string> - Overrides the "compatible" property of the dt root node
  dtb=<string>           - Linux kernel device tree file
  dump-guest-core=<bool> - Include guest memory in a core dump
  dumpdtb=<string>       - Dump current dtb to a file and quit
  firmware=<string>      - Firmware image
  graphics=<bool>        - Set on/off to enable/disable graphics emulation
  initrd=<string>        - Linux initial ramdisk file
  kernel=<string>        - Linux kernel image file
  mem-merge=<bool>       - Enable/disable memory merge support
  memory-backend=<link<memory-backend>> - Set RAM backendValid value is ID of hostmem based backend
  memory-encryption=<string> - Set memory encryption object to use
  memory=<MemorySizeConfiguration> - Memory size configuration
  phandle-start=<int>    - The first phandle ID we may generate dynamically
  smp=<SMPConfiguration> - CPU topology
  suppress-vmdesc=<bool> - Set on to disable self-describing migration
  usb=<bool>             - Set on/off to enable/disable usb

$ #https://github.com/yangxiaojuan-loongson/qemu-binary

```

```bash
$ #编译 QEMU_EFI.fd 参考https://github.com/tianocore/edk2-platforms/tree/master/Platform/Loongson/LoongArchQemuPkg
$ mkdir -p ~/.loongarch && cd ~/.loongarch
$ wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/CLFS-loongarch64-8.1-x86_64-cross-tools-gcc-glibc.tar.xz && tar -xvJf CLFS-loongarch64-8.1-x86_64-cross-tools-gcc-glibc.tar.xz
$ export PATH=~/.loongarch/cross-tools/bin:$PATH
$ mkdir -p ~/.loongarch/emulator-loongarch64
$ cd ~/.loongarch/emulator-loongarch64
$ export WORKSPACE=`pwd`
$ git clone https://github.com/tianocore/edk2.git
$ cd edk2
$ git submodule update --init
$ cd $WORKSPACE
$ git clone https://github.com/tianocore/edk2-platforms.git
$ cd edk2-platform
$ git submodule update --init
$ cd $WORKSPACE
$ git clone https://github.com/tianocore/edk2-non-osi.git
$ export PACKAGES_PATH=$WORKSPACE/edk2:$WORKSPACE/edk2-platforms:$WORKSPACE/edk2-non-osi
$ export GCC5_LOONGARCH64_PREFIX=loongarch64-unknown-linux-gnu-
$ source  $WORKSPACE/edk2/edksetup.sh
$ make -C edk2/BaseTools
$ build --buildtarget=DEBUG --tagname=GCC5 --arch=LOONGARCH64  --platform=Platform/Loongson/LoongArchQemuPkg/Loongson.dsc
$ #文件在~/.loongarch/emulator-loongarch64/Build/LoongArchQemu/DEBUG_GCC5/FV/QEMU_EFI.fd
$ #https://github.com/loongson/Firmware/blob/main/LoongArchVirtMachine/README_CN.md


```

```
#https://github.com/sunhaiyong1978/CLFS-for-LoongArch/blob/main/CLFS_For_LoongArch64.md
#https://github.com/emojifreak/qemu-arm-image-builder
#https://github.com/sunhaiyong1978/Yongbao
$ mkdir -p ~/.loongarch && cd ~/.loongarch
$ wget -c https://github.com/sunhaiyong1978/CLFS-for-LoongArch/releases/download/8.1/clfs-system-8.1-boot.loongarch64.squashfs && unsquashfs -user-xattrs clfs-system-8.1-boot.loongarch64.squashfs
$ wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/clfs-loongarch64-system-8.1-sysroot.squashfs && unsquashfs -user-xattrs clfs-system-8.1-sysroot.loongarch64.squashfs 
```

---

# ubuntu_x86_64下 交叉编译 loongarch64

### 0.下载并解压文件

```bash
mkdir -p ~/.loongarch && cd ~/.loongarch

wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/CLFS-loongarch64-8.1-x86_64-cross-tools-gcc-glibc.tar.xz && tar -xvJf CLFS-loongarch64-8.1-x86_64-cross-tools-gcc-glibc.tar.xz
wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/qemu-loongarch64 && chmod +x qemu-loongarch64
# 可选下载,运行时可能缺少动态库
wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/clfs-loongarch64-system-8.1-sysroot.squashfs && unsquashfs -user-xattrs clfs-system-8.1-sysroot.loongarch64.squashfs 
```

```			 	
#
#			tree -L 2 .loongarch/
#			.loongarch/
#			├── cross-tools
#			│   ├── bin
#			│   ├── include
#			│   ├── lib
#			│   ├── libexec
#			│   ├── loongarch64-unknown-linux-gnu
#			│   ├── share
#			│   └── target
#			├── qemu-loongarch64
#			└── squashfs-root
#			    ├── bin -> usr/bin
#    			    ├── boot
#			    ├── dev
#    			    ├── etc
#			    ├── home
#			    ├── lib -> usr/lib
#			    ├── lib64 -> usr/lib64
#			    ├── media
#			    ├── mnt
#			    ├── opt
#			    ├── proc
#			    ├── root
#			    ├── run
#			    ├── sbin -> usr/sbin
#			    ├── srv
#			    ├── sys
#			    ├── tmp
#			    ├── usr
#			    └── var
```


### 1. rust 1.72 添加 target loongarch64-unknown-linux-gnu

```bash
rustup self update
rustup update stable
rustup target add loongarch64-unknown-linux-gnu
```

### 2. 修改 ~/.cargo/config.toml
```
[target.loongarch64-unknown-linux-gnu]
linker = "loongarch64-unknown-linux-gnu-gcc"
runner = "qemu-loongarch64"
```

###  3.修改 ~/.bashrc
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
``` 
source ~/.bashrc				
loongarch.env
cargo new hello && cd hello
cargo run --target loongarch64-unknown-linux-gnu
```

``` 
source ~/.bashrc				
loongarch.env
git clone --depth=1 https://github.com/extrawurst/gitui
cd gitui
cargo update
cargo run --release --target loongarch64-unknown-linux-gnu
```

---
# 使用 llvm clang lld  rust1.72 clfs-loongarch64-system-8.1-sysroot.squashfs 编译loongarch64
### 0. 编译llvm core  , clang , lld

> ```bash
> mkdir -p ~/.loongarch/llvm
> 
> mkdir -p ~/.loongarch/llvm/build/llvm
> mkdir -p ~/.loongarch/llvm/install/llvm
> 
> mkdir -p ~/.loongarch/llvm/build/clang
> mkdir -p ~/.loongarch/llvm/install/clang
> 
> mkdir -p ~/.loongarch/llvm/build/lld
> mkdir -p ~/.loongarch/llvm/install/lld
> 
> cd ~/.loongarch/llvm
> 
> git clone --depth 1 https://github.com/llvm/llvm-project.git
> 
> #build llvm
> cmake -G Ninja -S ~/.loongarch/llvm/llvm-project/llvm -B ~/.loongarch/llvm/build/llvm -DLLVM_INSTALL_UTILS=ON -DCMAKE_INSTALL_PREFIX=~/.loongarch/llvm/install/llvm -DCMAKE_BUILD_TYPE=Release
> ninja -C ~/.loongarch/llvm/build/llvm install
> 
> #build clang
> cmake -G Ninja -S ~/.loongarch/llvm/llvm-project/clang -B ~/.loongarch/llvm/build/clang -DLLVM_EXTERNAL_LIT=~/.loongarch/llvm/build/llvm/utils/lit -DLLVM_ROOT=~/.loongarch/llvm/install/llvm -DCMAKE_INSTALL_PREFIX=~/.loongarch/llvm/install/clang -DCMAKE_BUILD_TYPE=Release
> ninja -C ~/.llvm_dev/build/clang install
> 
> #build lld
> cmake -G Ninja -S ~/.loongarch/llvm/llvm-project/lld -B ~/.loongarch/llvm/build/lld -DLLVM_EXTERNAL_LIT=~/.loongarch/llvm/build/llvm/utils/lit -DLLVM_ROOT=~/.loongarch/llvm/install/llvm -DCMAKE_INSTALL_PREFIX=~/.loongarch/llvm/install/lld -DCMAKE_BUILD_TYPE=Release
> ninja -C ~/.loongarch/llvm/build/lld install
>
>  cd ~/.loongarch && ln -s llvm/install llvm-18git
> 
> ```

### 1.下载qemu-loonarch64 和 sysroot
```bash
mkdir -p ~/.loongarch && cd ~/.loongarch
wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/qemu-loongarch64 && chmod +x qemu-loongarch64
wget -c https://github.com/loongson/build-tools/releases/download/2023.08.08/clfs-loongarch64-system-8.1-sysroot.squashfs && unsquashfs -user-xattrs clfs-system-8.1-sysroot.loongarch64.squashfs 
```

### 2. rust 1.72 添加 target loongarch64-unknown-linux-gnu

```bash
rustup self update
rustup update stable
rustup target add loongarch64-unknown-linux-gnu
```

### 3. 修改.bashrc
```bash
loongarch.cargo-sysroot(){
	export LOONGARCH_HOME=~/.loongarch
	export PATH=$LOONGARCH_HOME/llvm-18git/clang/bin:$LOONGARCH_HOME/llvm-18git/lld/bin:$LOONGARCH_HOME/llvm-18git/llvm/bin:$LOONGARCH_HOME:$PATH
	
	CC_loongarch64_unknown_linux_gnu="$LOONGARCH_HOME/llvm-18git/clang/bin/clang-18" \
	CFLAGS_loongarch64_unknown_linux_gnu="--sysroot=$LOONGARCH_HOME/squashfs-root" \
	AR_loongarch64_unknown_linux_gnu="$LOONGARCH_HOME/llvm-18git/llvm/bin/llvm-ar" \
	RANLIB_loongarch64_unknown_linux_gnu="$LOONGARCH_HOME/llvm-18git/llvm/bin/llvm-ranlib" \
	CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_LINKER="$LOONGARCH_HOME/llvm-18git/clang/bin/clang-18" \
	CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_AR="$LOONGARCH_HOME/llvm-18git/llvm/bin/llvm-ar" \
	QEMU_LD_PREFIX=$LOONGARCH_HOME/squashfs-root \
	CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_RUNNER="$LOONGARCH_HOME/qemu-loongarch64" \
	CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_RUSTFLAGS="-C link-arg=-fuse-ld=lld -C link-arg=--target=loongarch64-unknown-linux-gnu -C link-args=--sysroot=$LOONGARCH_HOME/squashfs-root -C target-feature=+crt-static" \
	CARGO_BUILD_TARGET="loongarch64-unknown-linux-gnu" cargo  $@
	 
}
```

### 4.测试
``` 
source ~/.bashrc				
cargo new hello && cd hello
loongarch.cargo-sysroot run 
```

``` 
source ~/.bashrc				
git clone --depth=1 https://github.com/extrawurst/gitui
cd gitui
cargo update
loongarch.cargo-sysroot build

file target/loongarch64-unknown-linux-gnu/debug/gitui
target/loongarch64-unknown-linux-gnu/debug/gitui: ELF 64-bit LSB executable, LoongArch, version 1 (SYSV), statically linked, for GNU/Linux 5.19.0, with debug_info, not stripped
```

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
ar = "loongarch64-unknown-linux-gnu-ar"
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
    
    echo "##############info##################"
    which loongarch64-unknown-linux-gnu-gcc
    which qemu-loongarch64
    echo LOONGARCH_SYSTEM_SYSROOT=$LOONGARCH_SYSTEM_SYSROOT
    echo QEMU_LD_PREFIX=$QEMU_LD_PREFIX
    echo LD_LIBRARY_PATH=$LD_LIBRARY_PATH
    echo LOONGARCH_HOME=$LOONGARCH_HOME   
}

loongarch.cargo(){
	if [ -z `which loongarch64-unknown-linux-gnu-gcc` ] 
	then
		loongarch.env
	fi
	
	CC_loongarch64_unknown_linux_gnu=$LOONGARCH_TOOLS_DIR/bin/loongarch64-unknown-linux-gnu-gcc \
	CXX_loongarch64_unknown_linux_gnu=$LOONGARCH_TOOLS_DIR/bin/loongarch64-unknown-linux-gnu-g++ \
	AR_loongarch64_unknown_linux_gnu=$LOONGARCH_TOOLS_DIR/bin/loongarch64-unknown-linux-gnu-gcc-ar \
	RANLIB_loongarch64_unknown_linux_gnu=$LOONGARCH_TOOLS_DIR/bin/loongarch64-unknown-linux-gnu-gcc-ranlib \
	CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNUN_LINKER=$LOONGARCH_TOOLS_DIR/bin/loongarch64-unknown-linux-gnu-gcc \
	CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNUN_RUNNER="qemu-loongarch64" \
	cargo  $@
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
cargo run --release --target loongarch64-unknown-linux-gnu
```



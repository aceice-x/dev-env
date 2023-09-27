# dev-env

### 测试
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

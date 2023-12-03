---
title: Rust 1.74 with OLLVM
date: 2023-12-03 15:30:00
categories:
  - Tech
tags: 
  - Rust
  - OLLVM
  - Obfuscation
excerpt: 虽然搞了很久，但这次终于感觉接近理想状态了
---

之前尝试过一次用 hikari 的 patch 去编译 rust 工具链，但是它似乎一编译大一点的 proj 就卡死了 hh。所以还是尝试去用原版的 ollvm 来搞一搞。

## 准备

这里使用 msys2 mingw64 toolchain 来进行编译，其安装方式不再赘述，可以参考 [msys2 官网](https://www.msys2.org/)。

```sh
# 拉取 rust 1.74.0 源码
git clone --single-branch --branch 1.74.0 https://github.com/rust-lang/rust
```

然后在 `.gitmodules` 中可以找到其对应的 llvm 版本信息：

![](https://cdn.silente.top/img/202312031555035.png)

```sh
# 接着拉取对应版本的 llvm 源码
git clone --branch rustc/17.0-2023-09-19 https://github.com/rust-lang/llvm-project.git
```

## 构建

这里采用 [DreamSoule/ollvm17](https://github.com/DreamSoule/ollvm17) 作为补丁。

直接将这个 repo 的文件夹去覆盖 llvm 源码，然后由 [issues 2](https://github.com/DreamSoule/ollvm17/issues/2) 可以知道需要更改一下 `llvm-project/llvm/include/llvm/IR/Function.h:722` 中 `BasicBlockListType` 和 `getBasicBlockList()` 的可见性，直接移动到 public 块下即可。

然后使用命令：

```sh
# config
cmake -G "Ninja" -S ./llvm-project/llvm -B ./build_dyn_x64 -DCMAKE_INSTALL_PREFIX=./llvm_x64 -DCMAKE_CXX_STANDARD=17 -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang;lld;" -DLLVM_TARGETS_TO_BUILD="X86" -DLLVM_INSTALL_UTILS=ON -DLLVM_INCLUDE_TESTS=OFF -DLLVM_BUILD_TESTS=OFF -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_BUILD_BENCHMARKS=OFF
# build
cmake --build ./build_dyn_x64 -j 16
# install
cmake --install ./build_dyn_x64
```

这里使用了 `ninja` 来并行编译，`-j` 来指定编译的并行数。

编译完成后，可以在 `./llvm_x64/bin` 中找到编译好的 llvm 工具链：

![](https://cdn.silente.top/img/202312031602253.png)

然后打开 rust 源码，将 `config.example.toml` 的内容给复制到 `config.toml` 中，然后修改其中的 `llvm-config` 为我们刚刚编译好的 llvm 工具链的路径，同时更改:

- `channel = "nightly"`，享受最新的 rust 特性
- `debug = false`，关闭 debug 信息，提高编译速度
- `[target.x86_64-unknown-linux-gnu]` -> `[target.x86_64-pc-windows-gnu]`，指定编译工具链


由于用的是 msys2 进行编译，看到一个 [issues](https://github.com/rust-lang/rust/issues/117567) 说这环境会出现一个独有的问题，幸好就在不久已经有人提供了一个 [draft pr](https://github.com/rust-lang/rust/pull/117833) 进行解决，经过尝试能够编译通过，所以就直接用了：

![](https://cdn.silente.top/img/202312031605829.png)



然后直接 `python x.py build`，在 `stage1` 中生成了这个版本的 rustc：

![](https://cdn.silente.top/img/202312031614190.png)

这里简单提一下 rust build 阶段，它一共分为三个阶段，有两种产物，其中 stage1 是由 stage0 提供的已发布的 rustc 来进行编译得到的编译器，而 stage2 是由自己 stage1 再编译自身从而得到的编译器，我们使用其实编译到 stage1 就够了。

接着编译 `cargo` 工具：

```sh
python x.py build tools/cargo
```


然后将 mingw64 的几个 dll 文件复制到 `stage1/bin` 下，`cargo.exe` 也放在这里：

![](https://cdn.silente.top/img/202312031641280.png)

llvm 也需要将除了 `libzstd.dll` 之外的 dll 文件复制到 `llvm/bin` 下。

在 msys2 外面测试：

![](https://cdn.silente.top/img/202312031643959.png)


接下来看看效果，使用 `cargo new hello-world` 新建一个项目，然后在 `.cargo/config.toml` 中加入：

```toml
[target.x86_64-pc-windows-gnu]
rustflags = [
    "-C", "llvm-args=-fla -bcf_loop=2 -split_num=2",
]
```

执行：

```sh
cargo build -r -Z build-std=panic_abort,std,core,alloc,proc_macro --target x86_64-pc-windows-gnu
```

同时执行一份不加混淆的：

![](https://cdn.silente.top/img/202312031716901.png)

去看一下 printf 源码，可以看到出现了 block split：

![](https://cdn.silente.top/img/202312031720320.png)

![](https://cdn.silente.top/img/202312031720600.png)

## 问题

不知道为啥现在编译出来的 cargo 工具链没办法在其他地方使用，它自己会去找 msys2 里的 linker，所以前面的测试也都是在 msys2 里编译的，虽然也不是不行，但是感觉不太优雅hh，看看后面有没有什么法子解决一下。


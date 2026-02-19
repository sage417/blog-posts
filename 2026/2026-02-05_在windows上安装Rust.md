---
title: '在windows上安装Rust'
date: '2026-02-05 00:00:00'
cover: 
toc: true
categories:
    - 代码
---
## 问题背景
在windows系统上安装Rust需要C++ 编译工具，正常使用Visual Studio安装工具将消耗大量空间和时间下载安装组件，这里提供另外一种安装流程，以最快速度在新电脑上成功安装并运行Rust代码

再次安装时发现一篇英文文档，描述如何安装 build tools 可以参考(windows-msvc)[https://rust-lang.github.io/rustup/installation/windows-msvc.html]

## 安装流程

1. 使用vs_BuildTools.exe安装C++ build tools，选择安装单个组件，选择MSVC v143 - VS 2022 C++ x64/x86 build tools (Latest)和Windows 11 SDK (10.0.22621.0)
2. Rust官网下载rustup-init.exe 进行安装
3. 使用rustup show来验证安装的组件信息

最终效果为
```
rustup show
Default host: x86_64-pc-windows-msvc


installed toolchains
--------------------
stable-x86_64-pc-windows-msvc (active, default)

active toolchain
----------------
name: stable-x86_64-pc-windows-msvc
active because: it's the default toolchain
installed targets:
  x86_64-pc-windows-msvc
```

## 最后小小吐槽
本地编译rust代码需要一台高性能电脑，并且最好在编译的同时干些别的事 :)
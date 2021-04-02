---
title: '使用Visual Studio Code阅读Switch Homebrew程序源码'
date: '2020-06-28 12:00:00'
cover: https://i.loli.net/2020/06/28/4eZCAUgk3WzfmHi.jpg
toc: true
categories:
    - DevKitPro
---

Visual Studio Code是当之无愧的代码利器，我看重的正是它独到的代码提示和Intellisense功能，这能有效帮助我阅读和学习别人编写的源码。而破解后的Switch上可以安装格式为“`nro`”的自制程序，它们使用C语言编写，为了方便阅读其源代码，我对VS Code进行如下配置。



## 第一步：搭建编译环境

DevKitPro组织维护着为任天堂系列主机做开发的工具集，这使得下载开发Switch上的程序更为简便。在[这里](https://devkitpro.org/wiki/devkitPro_pacman)下载包管理工具`dkp-pacman`，然后安装下列包：

```
# 安装
sudo dkp-pacman -S switch-dev
# 更新
sudo dkp-pacman -Syu
```

使用Root权限安装，默认会装到/opt/devkitpro。

如果MacOS 版本为 Catalina 会报错：

```
error: Partition / is mounted read only
error: not enough free disk space
error: failed to commit transaction (not enough free disk space)
```

因为Catalina 的根分区是只读的，所以可以用下面的命令指定安装位置:

```
sudo dkp-pacman -S switch-dev -r /System/Volumes/Data
```

## 第二步：配置Code

在Code中打开Homebrew程序的源码目录，按Ctrl+Alt+P呼出命令面板，找到“C/C++: Edit configurations”，点击打开C/C++扩展的配置文件“`c_cpp_properties.json`”。

然后将以下路径添加到“`includePath`”列表中。添加过程注意遵循JSON语法：

```json
{
    "configurations": [
        {
            "name": "switch",
            "includePath": [
                "${workspaceFolder}/**",
                "/opt/devkitpro/devkitA64/aarch64-none-elf/include/**",
                "/opt/devkitpro/libnx/include/**",
                "/opt/devkitpro/portlibs/switch/include/**"
            ],
            "defines": [],
            "macFrameworkPath": [
                "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks"
            ],
            "compilerPath": "/opt/devkitpro/devkitA64/bin/aarch64-none-elf-gcc",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64"
        }
    ],
    "version": 4
}
```

即刻生效。

## 第三步：测试

可以遵循以下的方法检验代码提示功能是否生效：

- 将鼠标移到任意一个对象名上，可以清楚看到它的定义；
- 按Ctrl键，将鼠标指针指向对象名或头文件名，它们会变成超链接，点进去即可跳转到它们的定义处。
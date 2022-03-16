---
title: 构建支持tf-lite的XiUOS系统并在QEMU虚拟机上运行
categories: IoT
date: 2022-03-16 20:53:25
tags:
cover: http://xuos.io/images/hero.svg
---


## 前置知识

[XiUOS](http://xuos.io/)是一款工业物联网操作系统；[tf-lite](https://www.tensorflow.org/lite)是[TensorFlow](https://www.tensorflow.org/)机器学习开源库的轻量化模型，可以将其神经网络模型部署到非桌面设备上；[QEMU](https://www.qemu.org/)是针对物联网系统的虚拟机环境。这篇文章是针对如何构建支持tf-lite的XiUOS系统并在QEMU虚拟机上运行的教程。

<!-- more -->

教程内使用的机器环境为

```text
Windows 10 Linux Subsystem
BUILD:    22000
BRANCH:   co_release
RELEASE:  Ubuntu 20.04.4 LTS
KERNEL:   Linux 4.4.0-22000-Microsoft
```

实际上，这篇教程完全适用于`Ubuntu 20.04 LTS`和`Ubuntu 18.04 LTS`，其他操作系统也可以根据命令进行适当修改。选择[WSL](https://docs.microsoft.com/en-us/windows/wsl/install)也是方便Windows用户完成环境配置，但是虚拟机性能可能会受到影响。

此外，完成配置还需要至少一款文本编辑器，教程内使用的是[VSCode](https://code.visualstudio.com/)，在这里不赘述其安装过程。读者也可以根据自己的习惯自行选择其他编辑器。

XiUOS可以在多种单片机环境下构建，这里使用的是`arm cortex-m0`，这也是tf-lite支持较好的环境，且QEMU在这种环境下的模拟对WSL的支持也较好。

> 经测试，编译`risc-v`架构的`hifive1`环境的XiUOS系统在WSL下的QEMU模拟时会占用CPU 100%并卡死，如果你使用的是WSL环境，请不要选择这一编译方式。

教程会从新安装好的WSL系统开始，到配置完成结束。

## 编译环境安装

首先需要安装必须的编译环境。在终端内输入

```bash
sudo apt update
sudo apt upgrade
```

更新Ubuntu软件包。然后输入

```bash
sudo apt install build-essential pkg-config cmake ninja-build git
sudo apt install libglib2.0-dev libpixman-1-dev
sudo apt install gcc make libncurses5-dev openssl libssl-dev bison flex libelf-dev autoconf libtool gperf libc6-dev
```

依次安装C/C++编译环境和git，用于后续XiUOS和QEMU源代码的下载和编译。

接下来使用终端移动到构建XiUOS系统的文件夹，例如

```bash
cd ~
mkdir testspace/XiUOS
cd testspace/XiUOS
```

然后下载安装裁减配置工具[kconfig-frontends](https://www.gitlink.org.cn/projects/xuos/kconfig-frontends)并安装ARM架构的编译工具链，使用以下命令

```bash
git clone https://git.trustie.net/xuos/kconfig-frontends.git
cd kconfig-frontends
./xs_build.sh
cd ..
sudo apt install gcc-arm-none-eabi
```

这样就完成了编译环境的安装。

## 下载编译QEMU

> 参考文档[Download QEMU](https://www.qemu.org/download/)。

根据文档，从gitlab仓库中下载并编译QEMU，使用以下命令

```bash
git clone https://gitlab.com/qemu-project/qemu.git
cd qemu
git submodule init
git submodule update --recursive
./configure
make
sudo make install
cd ..
```

其中`./configure`命令是配置了整个QEMU包，可能会导致安装过程较长。可以参考[相关博客](https://xyzzpwn.top/2020/09/10/ubuntu-bian-yi-an-zhuang-qemu/)中配置`./configure --target-list=XXX`的方法编译部分需要的内容。

安装完成后，就可以使用`qemu-system-XXX`的命令运行各种架构的虚拟机。例如可以使用`qemu-system-aarch64`运行arm64架构的虚拟机。

## 下载编译XiUOS

> 参考文档[从零开始构建矽璓工业物联操作系统：使用ARM架构的STM32F407-discovery开发板](http://xuos.io/doc/appdev/start_from_scratch/stm32f407-st-discovery.html)。

下载git仓库中的XiUOS源码，并用VSCode打开

```bash
git clone https://git.trustie.net/xuos/xiuos.git
cd xiuos
code .
```

打开的源码将在下一章中进行讲解，现在先来尝试编译源码中提供的`Hello World`样例。

> 本篇教程写作于`commit 3a35bee743`版本，该版本存在一个很大的漏洞，需要先进行修理，后续版本也可能对此进行维护，因此接下来一段代码修改请读者依照所处的编译环境自行选择。修改方法参考[这个issue](https://gitlink.org.cn/xuos/xiuos/issues/52079)。

接下来配置XiUOS

```bash
cd Ubiquitous/XiUOS/
make BOARD=cortex-m0-emulator menuconfig
```

运行成功将会进入如下配置界面：

![menuconfig](https://z4a.net/images/2022/03/16/menuconfig.png)

如界面介绍，界面的操作方法是按上下左右移动光标，`Y`表示开启功能，`N`表示关闭功能，`M`表示修改功能。XiUOS系统中内置了tf-lite库，需要在这里配置开启。具体位置在

```text
menuconfig
--->APP_Framework
--->Framework
--->support knowing framework
--->Tensorflow Lite for Micro
--->Using tensorflow lite for micro demo app
```

其他的配置项可以根据自己的需要进行添加。配置完成后将光标移到下方的`Save`按钮保存后到`Exit`按钮退出。

接下来就可以开始编译

```bash
make BOARD=cortex-m0-emulator
```

编译完成后应该可以看到类似下文的提示

```text
------------------------------------------------
link XiUOS_cortex-m0-emulator.elf
------------------------------------------------
   text    data     bss     dec     hex filename
  90604    1812    2728   95144   173a8 XiUOS_cortex-m0-emulator.elf
```

说明了编译生成的文件名为`XiUOS_cortex-m0-emulator.elf`，存放路径为当前目录下的`build`文件夹。尝试使用前文编译的QEMU虚拟机运行

```bash
qemu-system-arm -machine microbit -nographic -kernel build/XiUOS_cortex-m0-emulator.elf
```

如果出现了

```text
 _         _   _                  _          _ _
| |    ___| |_| |_ ___ _ __   ___| |__   ___| | |
| |   / _ \ __| __/ _ \ '__| / __| '_ \ / _ \ | |
| |__|  __/ |_| ||  __/ |    \__ \ | | |  __/ | |
|_____\___|\__|\__\___|_|    |___/_| |_|\___|_|_|

Build:       Mar 16 2022 17:33:52
Version:     3.0.5
Copyright:   (c) 2020 Letter

letter:$ Hello, world!
```

的提示，说明编译运行正常了。使用快捷键`ctrl-A`后按下`X`即可退出QEMU虚拟机。

## XiUOS源码逻辑简述

XiUOS文件目录为

```text
xiuos
├── APP_Framework
│   ├── Applications
│   ├── Framework
│   ├── Kconfig
│   ├── Make.defs
│   ├── Makefile
│   └── lib
├── Ubiquitous
│   ├── Nuttx
│   ├── RT_Thread
│   └── XiUOS
└── readme.md
```

`Ubiquitous`文件夹内存放的是操作系统的源码，因此我们前文的编译在`xiuos/Ubiquitous/XiUOS/`目录下进行。`APP_Framework`文件夹内存放的是在XiUOS系统中运行的程序代码，内涵的`Framework`文件夹存放的是程序运行的各种依赖，`Applications`文件夹存放的是应用程序，特别是`xiuos/APP_Framework/Applications/main.c`文件，是程序运行的入口。而`xiuos/APP_Framework/Framework/knowing/tensorflow-lite/tensorflow-lite-for-mcu`文件夹内则是存放有tf-lite的代码。因此在XiUOS内进行软件开发主要是修改作为入口的`main.c`文件实现需要的功能。

---
title: 通过国内镜像下载riscv工具链源码并自行编译
categories:
  - 经验和总结
tags:
  - riscv
abbrlink: 4b33f3aa
date: 2021-12-25 16:58:12
---

* 设置git选项避免失败;
* 通过gitee下载主仓;
* 修改.gitmodule为国内镜像,加速下载子仓;
* 编译rv64工具链并设置环境变量;

<!-- more -->

## 下载工具链源码

### 设置git选项

首先,设置`http.postBuffer`选项为`2000M`,我也不知道是什么作用,但我猜测他是一个库的文件大小的上限?

我设置成`1000M`的时候,一旦下载到大于1G的内容,就会显示出现异常的`EOF`.

```shell
git config --global http.postBuffer 2000M
```

### 下载代码

这个命令只是下载了主仓,如果你想要同时递归下载所有子仓的话,可以加上参数`--recursive`.也可以下载完主仓之后再另外下载子仓.

```shell
git clone https://github.com/riscv/riscv-gnu-toolchain
```

`cd`进入源码目录后,输入以下命令下载子仓,其中到`--progress`命令可以显示下载进度.

```shell
git submodule update --init --recursive --progress
```

### 加速下载

如果下载实在很慢,可以修改源码目录下的`.gitmodules`文件,

其中的github仓库,在gitee都有镜像.只需要把地址改成`https://gitee.com/mirrors/<仓库>`

比如`https://github.com/riscv-collab/riscv-binutils-gdb.git`可以改成`https://gitee.com/mirrors/riscv-binutils-gdb.git`

其中`qemu`在`gitee`上没有镜像,但是北京外国语大学提供了镜像.我们可以把那一项的地址改成`url = https://mirrors.bfsu.edu.cn/git/qemu.git`

修改完上述内容之后,先执行以下命令刷新,然后再去下载子仓.

```shell
sudo git submodule sync
```

## 编译

在编译之前,需要安装一些工具

```shell
sudo apt install autoconf automake autotools-dev curl python3  libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex  texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev
```

在`/opt`目录下创建一个`riscv`目录,用以存放工具链.

```shell
mkdir /opt/riscv
```

开始编译,以下命令编译的是64位,如果想要编译其他版本,可以看`readme`文档的说明来设置对应参数.

```shell
./configure --prefix=/opt/riscv
make 
```

编译完成后,`/opt/riscv/bin`中会出现一些可执行文件.

为了方便调用他们,可以修改`~/.bashrc`脚本,每次启动终端时自动将这个目录添加到环境变量.

在`~/.bashrc`的最末尾,加上这句话.

```shell
export PATH=$PATH:/opt/riscv/bin
```

如果你的工具链放在其他位置,那么就把上文到`/opt/riscv/bin`改成你的工具链的可执行文件的路径即可.

这时候重新启动终端,就可以直接在命令行使用你编译的riscv工具链.
---
title: linux下使用nuclei-sdk编译longan-nano程序并获取串口打印信息
categories:
  - 经验和总结
tags:
  - riscv
  - longan
  - nuclei
  - gd32vf103
abbrlink: 213a381f
date: 2021-12-25 19:05:04
---

* 下载nuclei sdk及其他工具;
* 编译helloworld程序;
* dfu下载可执行文件;
* usb-jtag使用
* minicom接收打印信息;

<!-- more -->

## 准备nuclei-sdk

### riscv工具链

从[nuclei官网](https://www.nucleisys.com/download.php)下载riscv gnu toolchain.

选择`centos/ubuntu x86-64`版本.

把里面的`gcc`目录解压,某个目录(暂时把这个目录记做`dir1`)

### openocd工具

从[nuclei官网](https://www.nucleisys.com/download.php)下载openocd.

选择`centos/ubuntu x86-64`版本.

解压之后目录结构如下

```
Nuclei
|- openocd
   |- 0.10.0-15
      |- bin
      |- ...
```

首先我们需要调整他的目录,把`0.10.0-15/`里的文件都直接移到`openocd/`.去掉`0.10.0-15/`这一级目录.

然后把`openocd`也移动到上文的`dir1`.

现在`dir1`的街镇应该如下:

```
dir1
|- gcc
   |- bin
   |- ...
|- openocd
   |- bin
   |- ...
```

### dfu-util工具

我们使用`dfu-util`工具将程序刷入`longan nano`.

这里不要使用官网到版本,也不要apt安装

他们可以刷入,但无效果,[详情见](https://bbs.21ic.com/icview-3069966-1-1.html)

建议从[这个网站](https://sourceforge.net/projects/dfu-util/files/)下载最新的[dfu-util-0.11-binaries.tar.xz](https://sourceforge.net/projects/dfu-util/files/dfu-util-0.11.tar.gz/download)

解压后,里面有一个目录`linux-amd64`,其中有我们需要的4个可执行文件.

直接在命令行用`.绝对路径/dfu-util`的方式可以执行他.

如果你嫌麻烦的话,可以把它加到环境变量,在`~/.bashrc`文件末尾加上一行.

```shell
export PATH=$PATH:<你的dfu-util所在目录的绝对路径>
```

加上这一行重新打开终端,就可以直接使用`dfu-util`.



### nuclei源码

从gitee下载nuclei-sdk源码

```shell
git clone https://gitee.com/Nuclei-Software/nuclei-sdk
```

## 编译

### 设置工具目录

进入`nuclei-sdk`,里面有一个脚本`setup.sh`.

```shell
NUCLEI_TOOL_ROOT=~/Nuclei
## Create your setup_config.sh
## and define NUCLEI_TOOL_ROOT like below
## NUCLEI_TOOL_ROOT=/home/develop/Software/Nuclei
SETUP_CONFIG=setup_config.sh

[ -f $SETUP_CONFIG ] && source $SETUP_CONFIG

[ -f .ci/build_sdk.sh ] && source .ci/build_sdk.sh
[ -f .ci/build_applications.sh ] && source .ci/build_applications.sh

echo "Setup Nuclei SDK Tool Environment"
echo "NUCLEI_TOOL_ROOT=$NUCLEI_TOOL_ROOT"

export PATH=$NUCLEI_TOOL_ROOT/gcc/bin:$NUCLEI_TOOL_ROOT/openocd/bin:$NUCLEI_TOOL_ROOT/qemu:$PATH
```

观察发现,他会从`setup_config.sh`文件获取`SETUP_CONFIG`的值,如果获取不到就用`~/Nuclei`(这显然是个错误的路径).

获取之后,他就会在`$SETUP_CONFIG`目录寻找工具链和`openocd`.

所以这里我们需要把它设置成上文所说到`dir1`.

所以我们需要在`nuclei-sdk`中创建一个文件`setup_config.sh`,内容如下:

```shell
NUCLEI_TOOL_ROOT=<你的dir1的绝对路径>
```

### 设置环境

写好上面的脚本之后,每次进入`nuclei-sdk`目录编译之前都要运行`setup.sh`脚本.

```shell
source setup.sh
```

运行之后,就可以直接使用`riscv-nuclei-elf-gcc`等工具.

### 编译helloworld

使用`make`,编译`application/baremetal/helloworld`目录下的程序.参数需要设置`SOC`类型和板类型.

命令如下

```shell
make PROGRAM=application/baremetal/helloworld SOC=gd32vf103 BOARD=gd32vf103c_longan_nano all
```

执行之后会在`application/baremetal/helloworld`产生一个`helloworld.elf`文件.

使用以下命令,把它编成`helloworld.bin`.

```shell
make PROGRAM=application/baremetal/helloworld SOC=gd32vf103 BOARD=gd32vf103c_longan_nano bin
```

## 下载

### 安装驱动

在下载和运行之前,为了使`USB`工作正常,我们也许需要安装以下软件.

也可能不需要,但还是装吧.

```shell
sudo apt install libusb-0.1-4 libftdi1 libhidapi-hidraw0
```

### dfu查看设备列表

通过`typec`口将电脑和`longan nano`连接.

`longan nano`上电之后,按住`boot`,同时按一下`reset`,松开`reset`之后再松开`boot`.

这时`longan nano`就会进入下载模式,等待刷入程序.

执行命令`dfu-util -l`查看当前已连接设备,你会发现权限不足.

先用`su root`切换到`root`用户.

注意`deepin`的`root`用户默认密码是空,你需要`sudo passwd`设置密码之后才能登录.

进入`root`之后,再次执行`dfu-util -l`,正常来说可以显示你插入到设备.

如果你的还是没有显示,那就不知道为什么了.

### dfu下载程序

惨案

输入以下命令下载程序到`longan nano`

```shell
sudo ./dfu-util -D <你的bin文件路径> -s 0x08000000
```

一阵摇晃之后,程序就下载好了,这时候可以按`reset`启动.

## 运行

由于这个程序是通过串口打印一些信息,我们要用电脑连到串口,来得知他到底有没有正常打印.

这时候就需要连接`jtag`.

### jtag接线

调试器和`longan nano`的连接方式如下:

| 调试器 | longan | 
| --- | --- |
| GND |	GND |
| RXD |	T0 |
| TXD |R0 |
| TDI | JTDI |
| RST |	RST |
| TMS | JTMS |
| TDO | JTDO |
| TCK | JTCK |


这里有3个问题:

1. 网上的旧版本外壳错误

在网上找到的大部分`sipeed jtag`图片,外壳上标有两个`GND`和一个`NC`.

但我手上的却只有一个`GND`,并且有`3.3V`和`5V`.这让我非常困惑.

直到后来搜到一个[博客](https://k4zukdiyelec.blogspot.com/2019/09/sipeed-tang-primer-hummingbird.html).

貌似是网上的版本标记错了,我的才是正确的.

2. 调试器上的引脚和外壳标记的引脚如何对应?

可以观察到引脚那一圈塑料有一个缺口,外壳的表格在`TMS`位置有一个记号.

显然有缺口的位置是`TMS`,这样就能确定调试器上的引脚分别是什么引脚了.

3. longan nano的RST在哪?

他不在尾部,看文档可以得知,他在typec接口的旁边.

最终接线如图

![](/images/20211225192004.png)

### 连接电脑

安装`minicom`.

```shell
sudo apt install minicom
```

把`jtag`调试器插入电脑,同时用`typec`给`longan nano`供电.

这时候在`/dev`目录会出现`ttyUSB0`.

这时候使用`minicom`连接`/dev/ttyUSB1`,命令如下:

```shell
sudo minicom -D /dev/ttyUSB1 -b 115200
```

### 输出结果

打开`minicom`之后,按`longan nano`的`reset`,会重新执行程序并且在终端打印一些信息.

如果你没有`ttyUSB1`或者没有输出,我觉得插口接触不良是很大的可能.

![](/images/20211225193027.png)

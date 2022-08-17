---
title: Nokia N1刷惜旧官改ROM
categories:
  - 经验和总结
tags:
  - nokia n1
  - 惜旧
  - android
abbrlink: 4f38d4d9
date: 2022-01-28 16:48:29
---

* 解锁bootloader;
* 刷入twrp;
* 刷入ROM;

<!-- more -->

## 声明

十分鸣谢制作惜旧ROM的大佬(QQ群:570160568).

更多相关文件可以进群下载,如果群满人也可以从这个[onedrive链接下载](https://1drv.ms/u/s!AlAdPSrz4UaPjGHq01n1flodaY8c?e=gaFuqD).

ROM带有以下软件

> google设置
> 黑域 补丁版
> MT管理器
> nova设置
> play商店
> QQ输入法
> 设置
> 时钟
> superSU
> 图库
> via
> 网速指示器
> wps office
> 下载
> google相机 

## 准备工作

### 安装adb和fastboot

```shell
sudo apt install adb fastboot
```

### 下载所需的文件

| 文件 | 作用 |
| --- | --- |
| `twrp2.8.7.1-nokia-n1-zh-cn.img` | recovery文件 |
| `Nokia_n1-B19(5.1)-惜旧新官改-MD5_c31812148c.zip` | ROM包 |

[下载链接](https://1drv.ms/u/s!AlAdPSrz4UaPjGHq01n1flodaY8c?e=gaFuqD).

## 进入twrp

首先打开`开发者模式`,在`开发者模式`打开`USB调试`和`oem解锁`.



弹出`允许USB调试吗?`,选择`一律允许使用这台计算机进行调试`,然后`确认`.

输入以下命令进入`bootloader`.

```shell
adb reboot bootloader
```

输入以下命令解锁`bootloader`

`linux`需要使用`sudo`,其他系统可以不写`sudo`.

```shell
sudo fastboot oem unlock
```

输入以下命令刷入`twrp`

```shell
sudo fastboot flash boot <你存放的路径>/twrp2.8.7.1-nokia-n1-zh-cn.img
```

重启,就会进入`twrp`.

## 刷入惜旧ROM

在`twrp`中,选`清除->高级`,选择全部选项,滑动确认清除.

选`清楚->格式化data`,滑动确认清除.

### 方法1 线刷

选`高级->adb线刷`,确认开始.

电脑输入命令

```shell
adb sideload <你存放的路径>/Nokia_n1-B19(5.1)-惜旧新官改-MD5_c31812148c.zip
```

### 方法2 卡刷

有可能你用`adb sideload`传不进去.

但是你的电脑识别到平板了,可以把`ROM`包传到平板上.

那么你可以直接在`twrp`内找到你传进去的包,然后直接安装,图形化得很.

一阵摇晃之后,就会进入系统.

第一次有点久,不要担心.
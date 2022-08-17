---
title: 路由器ttl连接方法
categories:
  - 经验和总结
tags:
  - ttl
  - minicom
  - tftp
  - openwrt
  - uboot
abbrlink: edb033a1
date: 2022-01-06 13:47:35
---

* 安装软件;
* 基本配置;
* 排针焊接和连线;

<!-- more -->

## 安装minicom和tftpd-hpa

### 安装软件

```shell
sudo apt install minicom
sudo apt install tftpd-hpa
sudo apt install tftp
```

### 配置tftp

查看`tftpd-hpa`状态.

```shell
sudo service ftfpd-hpa status
```

备份配置文件

```shell
sudo cp /etc/default/tftpd-hpa /etc/default/tftpd-hpa.bak
```

编辑`/etc/default/tftpd-hpa`文件.

修改`TFTP_OPTIONS`的值改为`--create`

```shell
TFTP_OPTIONS="--secure --create"
```

修改`TFTP_DIRECTORY`的的值为你想要分享的目录(绝对路径),例如

```shell
TFTP_DIRECTORY="/home/zzidun/tftp-file"
```

这个目录你要先创建出来.

修改那个目录的权限.

```shell
sudo chown -R 777 /home/zzidun/tftp-file
```

重启`tftp`服务.

```shell
sudo service tftpd-hpa restart
```

## 电脑终端打开路由器命令行

### 连接

拆开路由器,可能会找到一排孔,上面标有`GND TX RX 3.3V`等.

给这四排孔焊上针脚,就像这样:

然后接到`ttl`转`usb`的玩意上(比如淘宝上买的`ch340`).接线方法如下:

| 路由器 | 转接器 |
| --- | --- |
| GND | GND |
| TX | RX |
| RX | TX |

注意,`3.3v`不要接.

把任意`LAN`口用网线接到电脑,在电脑上设置有线网口`ip`为`192.168.1.2`,掩码为`255.255.255.0`,网关为`192.168.1.1`.

### minicom

打开minicom

```shell
sudo minicom -s
```

一定要`sudo`不然找不到设备的.

这时候进入一个界面:

```
    +-----[configuration]------+
    | Filenames and paths      |
    | File transfer protocols  |
    | Serial port setup        |
    | Modem and dialing        |
    | Screen and keyboard      |
    | Save setup as dfl        |
    | Save setup as..          |
    | Exit                     |
    | Exit from Minicom        |
    +--------------------------+
```

选择`Serial port setup`.

```
    +-----------------------------------------------------------------------+
    | A -    Serial Device      : /dev/ttyUSB0                              |
    | B - Lockfile Location     : /var/lock                                 |
    | C -   Callin Program      :                                           |
    | D -  Callout Program      :                                           |
    | E -    Bps/Par/Bits       : 115200 8N1                                |
    | F - Hardware Flow Control : Yes                                       |
    | G - Software Flow Control : No                                        |
    |                                                                       |
    |    Change which setting?                                              |
    +-----------------------------------------------------------------------+
```

按`a`跳到`Serial Device`选项,改为你的`usb`设备,一般是`/dev/ttyUSB0`,回车确认.

按`e`跳到`Bps/Par/Bits`选项,设置波特率为`115200`(通常是).

按回车一路确认回到第一个菜单,选择`Save setup as dfl`保存为默认配置.

然后选择`Exit`退出设置.

直接`sudo minicom`就连上了.
---
title: 'deepin开启远程桌面服务,使用vnc和rdp连接deepin'
categories:
  - 安装和配置
tags:
  - deepin
  - vnc
  - rdp
  - xrdp
abbrlink: '70069604'
date: 2021-12-25 17:42:56
---

* 安装openssh;
* 安装和配置x11vnc;
* 安装和配置xrdp;
* 设置开机启动;

<!-- more -->

要注意的是,以下操作对于其他打debian系发行版并不通用.

## 安装和启动sshd

```shell
sudo apt install openssh-server
sudo systemctl enable ssh.service
```

## 安装和配置x11vnc

```shell
sudo apt install x11vnc
```

设置密码
```shell
x11vnc -storepasswd
```
输入两次密码然后输入`y`确认把密码相关文件保存到文件`/home/<用户名>/.vnc/passwd`

拷贝上述文件到目录`/etc/x11vnc.pass`

```shell
sudo cp /home/<用户名>/.vnc/passwd /etc/x11vnc.pass
```

然后编辑文件`/lib/systemd/system/x11vnc.service`

内容如下
```
[Unit]
[Unit]
Description=Start x11vnc at startup.
After=multi-user.target
[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -auth guess -forever -loop -noxdamage -repeat -rfbauth /etc/x11vnc.pass -rfbport 5900 -shared
[Install]
WantedBy=multi-user.target
```

上面的`ExecStart`一项,里面有一个`/etc/x11vnc.pass`,这个是前面到密码文件的路径,你也可以不拷贝到`/etc`目录,而是直接设置成`/home/<用户名>/.vnc/passwd`

## 设置开机启动

```shell
sudo systemctl enable x11vnc.service
```

这时候重启,就可以用vnc软件连接到电脑了.

## 安装xrdp并配置开机启动

(需要先完成以上步骤

```shell
sudo apt install xrdp
sudo systemctl enable xrdp
```

重启之后就可以用windows到rdp客户端连接到电脑了

注意选择`VNC-any`协议,端口选择`5900`
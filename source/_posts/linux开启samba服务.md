---
title: linux开启samba服务
categories:
  - 安装和配置
tags:
  - linux
  - samba
abbrlink: db95d7df
date: 2022-01-27 15:06:24
---

* samba安装;
* 添加用户;
* 配置文件;

<!-- more -->

## 安装samba

```shell
sudo apt install samba
```

## 添加一个samba用户

```shell
useradd <用户名>
```

使用`useradd`创建的是一个无密码,无主目录的用户.

如果使用`adduser`,会创建主目录等等.

使用`smbpasswd`命令设置这个用户登录`samba`的密码

```shell
smbpasswd -a <用户名>
```

## 设置共享的文件夹

给文件夹设置`777`权限.

```shell
chmod 777 <路径>
```

## 修改配置文件

修改`/etc/samba/smb.conf`文件,每当想要共享一个文件夹,就在后面添加内容如下:

```shell
[<共享文件夹在客户端中显示的名称>]
comment = <注释>
browseable = yes
path = <共享文件夹路径>
create mask = 0777
directory mask = 0777
valid users = <登录这个文件夹用的用户名>
public = no
available = yes
writeable = yes
```

例如,我们想要直接共享用户`zzidun`的主目录,`samba`登录的用户为`smb`.

那么就编写配置文件如下:

```shell
[desktop-home]
comment = zzidun-share
browseable = yes
path = /home/zzidun
create mask = 0777
directory mask = 0777
valid users = smb
public = no
available = yes
writeable = yes
```

## 重启服务

```shell
service smbd restart
```

之后就可以在局域网内的各个设备通过`ip`访问这个文件夹.
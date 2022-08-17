---
title: linux桌面使用中的那些事
categories:
  - 安装和配置
tags:
  - linux
  - anbox
abbrlink: 17eed1d
date: 2021-12-28 20:42:48
---

* 安装时分区注意;
* 使用tmpfs;
* 配置vnc和xrdp远程;
* 快捷方式;
* 应用商店报错修复;
* 常用软件安装(go,mariaDB,npm,firefox,安卓模拟器);

<!-- more -->

## 制作启动光盘

```
wodim -sao -v -speed=1 dev=/dev/<光盘设备> <iso文件>
```

我留意到直接挂载是不能写入的,需要用wodim工具.


## 安装时分区注意

不要用默认分区,不要用默认分区,不要用默认分区.

比如,`deepin`默认分区只给根目录分配`15G`空间,一下子就满了,满了很难搞,非常折磨.

我的通常分区方式如下

| 挂在目录 | 文件系统 | 大小 |
| --- | --- | --- |
| `/boot/efi` | `fat32` | `100M` |
| `/` | `ext4` | `其余全部空间` |
| `<无>` | `linux-swap` | `8G` |

以前我把`/boot`独立分区弄得很小(`400M`),然后某次更新竟然满了,做什么都失败.

只能进`live`里面用`gparted`改分区.

我其实觉得`/boot`分区不用单独分出来.

我心里觉得除非是某些目录需要特殊的文件系统比如`efi`需要`fat32`,否则同硬盘分区就是扯蛋:

1. 文件跨分区移动耗时很长,像是跨硬盘移动一样,哪怕他们实际上在同一个硬盘.
2. 常常因为分区不合理,导致某一分区满了,其他分区还有很多空间,这时候调整非常麻烦.
3. 实际上用文件夹就能取得和分区一样的效果,而且没有上面的缺点.
4. 同一个硬盘的分区不会有什么保险的作用,硬盘损坏依旧是所有分区遭殃.

可见,我们根本没有理由去划分他.

## 移动/tmp目录到tmpfs

`/tmp`目录是缓存文件的目录,开关机就会被系统删除.

这东西显然应该放到内存,还能更快存取.实在没有必要放到硬盘.

编辑`/etc/fstab`文件,后面加上以下内容

```shell
tmpfs /tmp tmpfs defaults 0 0
tmpfs /var/tmp tmpfs defaults 0 0
```

## 配置远程桌面

{% post_link deepin开启远程桌面服务-使用vnc和rdp连接deepin %}

其他系统可以看这个[链接](https://www.hiroom2.com/ubuntu-2004-xrdp-kde-en)

## 指定软件版本

有时候我们可以同时安装多个版本的`gcc`,这时候可以使用`update-alternatives`命令来修改默认使用的`gcc`.

```shell
sudo update-alternatives --install /usr/bin/gcc gcc <需要使用的gcc的路径> <足够大的数值>
```

这个足够大的数值只要确保比之前指定过的数值大就可以,从来没有指定过就设置为`1`也行.

## 快捷方式

### 制作快捷方式

当我们有一个可执行文件,希望能够在开始菜单直接打开他.

首先可以编写一个文件`<文件名>.desktop`,来制作快捷方式.

```shell
[Desktop Entry]
Exec=<可执行文件地址>
Name=<快捷方式显示的名称>
Icon=<网上找的图标.svg>
Type=Application
```

以上都是绝对路径,比如

```shell
[Desktop Entry]
Name=Uengine
Exec=/home/zzidun/app/uengine/uengine.sh
Icon=/home/zzidun/app/uengine/anbox.svg
Type=Application
```

### 快捷方式移动到开始菜单

复制到`/usr/share/applications`目录即可.

```shell
sudo cp <xxx.desktop> /usr/share/application/
```

## 应用商店安装到75%报错

一般是由于卸载软件没有完成导致的,用以下命令可以解决:

```shell
sudo apt-get --fix-broken install -y
```

## apt install XXX-

千万谨慎,不要在安装命令后面多写一个横杆`-`,这代表卸载.

某一次我想要看看软件源提供了哪些版本的`gcc`,于是输入了`sudo apt install gcc-`,然后按`tab`查看列出的软件包.

这时候列出的软件包列表是用回车翻页的,我一直按回车,没想到多按了几个.

然后就变成执行了`sudo apt install gcc-`,一堆软件被卸载了,我人也裂开了.

## 休眠设置

看这个[文章](https://xie.infoq.cn/article/af26942709c82ab1c7cd47f87)

和这个[文章](https://blog.csdn.net/defrag257/article/details/103997722)


## 常用软件安装

### git设置

```shell
git config --global user.email "<邮箱>"
git config --global user.name "<用户名>"
git config --global http.postBuffer 10000M
```

### go

```shell
wget https://dl.google.com/go/go1.13.linux-amd64.tar.gz

sudo tar -C /usr/local -xzf go1.13.linux-amd64.tar.gz
```

然后修改文件`~/.bashrc`

加上这一句
```shell
export PATH=$PATH:/usr/local/go/bin
```

然后
```shell
source ~/.bashrc
```

检查是否安装完成

```shell
go version
```

### npm
{% post_link 自行安装nodejs和npm-解决apt源版本不兼容问题 %}

### mariaDB

{% post_link 安装和配置mariaDB %}

### firefox

缩小顶边栏

```
顶部右键 -> Customisze Firefox -> 屏幕靠下TitleBar取消打勾
```

使用内存作为缓存

火狐地址栏输入`about:config`,进入配置页面.

搜索栏输入`browser.cache.disk.enable`设置为`false`.


搜索栏输入`browser.cache.memory.enable`设置为`true`.


搜索栏输入`browser.cache.memory.capacity`设置为`-1`.

触屏

修改`/etc/security/pam_env.conf`,在最后增加一行`MOZ_USE_XINPUT2 DEFAULT=1`
在Firefox中，地址栏输入`about:config`，修改`dom.w3c_touch_events.enabled`为`1`。最后重启电脑。

### platformIO

直接`vscode`安装对应插件.

重新启动`vscode`.

默认情况下,platformIO会把新建的项目放到C盘用户文件里面某个很深的路径.

修改新建项目时保存路径(在pio的终端输入)

```shell
pio settings set projects_dir <路径>
```

### minicom和fastboot找不到设备

他们需要`sudo`.

### anbox

官方的教程中,找不到`binder`是正常的,`ubuntu20`之后都找不到的.

接着下一步就好.

`anbox.appmgr`启动不了,显示以下错误,是需要设置环境变量.

```shell
[ 2022-01-06 05:29:58] [launch.cpp:168@operator()] Session manager is not yet running, trying to start it
[ 2022-01-06 05:29:58] [launch.cpp:117@launch_session_manager] Started session manager, will now try to connect ..
[ 2022-01-06 05:29:59] [splash_screen.cpp:55@SplashScreen] Window has no associated renderer yet, creating one ...
[ 2022-01-06 05:30:04] [daemon.cpp:61@Run] [org.freedesktop.DBus.Error.ServiceUnknown] The name org.anbox was not provided by any .service files
```

只需要在`~/.bashrc`文件末尾加上这个:

```shell
export EGL_PLATFORM=x11
```

重新打开终端,输入`anbox.appmgr`就可以打开了.

### steam

需要安装以下软件

```shell
sudo apt install libgl1-mesa-dri:i386 libgl1:i386
```

`~/.profile`末尾加上变量

```shell
PROTON_USE_WINED3D=1
```

### kvm

以下方法转载自[这个博客](https://linux.cn/article-11151-1.html)

打开`/etc/libvirt/libvirtd.conf`进行编辑：

```shell
    sudo vi /etc/libvirt/libvirtd.conf
```

将`UNIX`域套接字组所有者设置为`libvirt`：

```shell
unix_sock_group = "libvirt"
```

调整`UNIX`域套接字的读写权限：

```shell
unix_sock_rw_perms = "0770"
```

步骤 3：启动并启用`libvirtd`服务

```shell
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```

添加用户到`libvirt`组

```shell
sudo usermod -a -G libvirt $(whoami)
```

### qbittorent增强版

可以防止迅雷吸血.

迅雷得到网上其他bt软件的用户分享的资源,却不给别人分享.

这样自私自利的软件需要我们一起抵制.

下面命令安装的版本,可以禁止向迅雷,qq等吸血软件分享.

```cpp
sudo add-apt-repository ppa:poplite/qbittorrent-enhanced
sudo apt-get update
sudo apt install qbittorrent-enhanced
```

`工具->首选项->高级->上传连接策略->反吸血`
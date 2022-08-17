---
title: '自行安装nodejs和npm,解决apt源版本不兼容问题'
categories:
  - 安装和配置
tags:
  - vue
  - nodejs
  - npm
  - debian
  - deepin
abbrlink: 917b256a
date: 2021-12-25 17:40:39
---

* 安装nvm;
* 使用nvm安装新版本nodejs和npm

<!-- more -->

debian系发行版有一个逆天的地方,那就是apt安装的nodejs很老,npm却很新.

它们居然tmb不匹配.

想要安装自己需要的版本可以这样做.

安装nvm

```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
```

查看所有nvm版本,以下命令会列出所有版本

```shell
nvm ls-remove
```

安装想要的版本

```shell
nvm install v12.22.6
```

查看当前使用的nodejs版本.箭头指着的就是现在用的

```shell
nvm ls
```

切换当前使用的nodejs版本.

```shell
nvm use v[版本号]
```

设置默认使用的版本

```shell
nvm alias default [版本号]
```

安装cnpm

```shell
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

安装vue-cli

```shell
npm install -g @vue/cli
# 或者 cnpm install -g @vue/cli
```
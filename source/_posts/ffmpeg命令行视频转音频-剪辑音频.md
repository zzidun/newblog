---
title: ffmpeg命令行视频转音频 剪辑音频
categories:
  - 经验和总结
tags:
  - ffmpeg
  - bilibili
abbrlink: ca9c0f8d
date: 2021-12-27 21:58:56
---

* ffmpeg命令行视频转音频
* 简单剪辑

<!-- more -->

首先推荐一个[下载B站视频的网站](https://bilibili.iiilab.com/)

当我们想要把某些up主音乐视频保存到本地方便之后听的时候.

可以从上面下载下来.

### 视频转音频

```shell
ffmpeg -i <视频文件.mp4> <音频文件.mp3>
```

一阵摇晃之后,就会产生一个音频文件.

### 音频的简单剪辑

我们通常需要剪出其中的一段.


可以这样,其中开始位置结束位置的格式形如`时:分:秒.百分之一秒`,比如`00:01:02.33`.

```shell
ffmpeg  -i <音频文件.mp3>  -vn -acodec copy -ss <开始位置> -t <结束位置> <输出音频文件.mp3>
```

拼接两个音频

```shell
ffmpeg -i "concat:<音频文件1.mp3>|<音频文件2.mp3>" -acodec copy <音频文件3.mp3>
```
---
title: C++的各种宏和属性
categories:
  - 经验和总结
tags:
  - C++
  - 细说std源码
abbrlink: 3b391a6f
date: 2022-02-01 11:33:18
---

* 总结中;
* 宏;
* progma;
* \_\_cpluscplus;
* \_\_attribute\_\_;

<!-- more -->

## 宏

| 宏名称 | 值 | 含义 |
| --- | --- | --- |
| `_GLIBCXX17_DEPRECATED` | | 这个东西即将被删除,不推荐使用 |
| `_GLIBCXX_GTHREAD_USE_WEAK` | 对于`MINGW32`为`0`,否则为`1` | 不知道 |
| `_GLIBCXX_HAS_GTHREADS` | 如果可用则为`1` | 表示当前`gthread`库是否可用 |
| `_GLIBCXX_HIDE_EXPORTS` | | 在`gthr.h`中,如果没有定义这个宏,就会执行`#pragma GCC visibility pop` |
| `_GLIBCXX_VISIBILITY` | `__attribute__ ((__visibility__ (#V)))` | 改变可见性 |
| `_GLIBCXX_HAVE_ATTRIBUTE_VISIBILITY` | 如果可以设置可见性,就有定义 | 决定`_GLIBCXX_VISIBILITY`是否起作用 |


## pragma

常常见到`#pragma xxx yyy`的用法,总结如下:


### system_header

```cpp
#pragma GCC system_header
```
像对待系统库那样对待这个文件(也就是这个文件可能不符合标准,但是不要警告).

### visibility

指定目标文件外部链接实体的可见性.

```cpp
#pragma GCC visibility push (value)
```

`value`可以为以下值

default
    实体在共享库中可见,并且可以被抢占.
protected
    Indicates that the affected external linkage entities have the protected visibility attribute. These entities are exported in shared libraries, but they cannot be preempted.
hidden
    Indicates that the affected external linkage entities have the hidden visibility attribute. These entities are not exported in shared libraries, but their addresses can be referenced indirectly through pointers.
internal
    Indicates that the affected external linkage entities have the internal visibility attribute. These entities are not exported in shared libraries, and their addresses are not available to other modules.

可以通过以上指令来指定实体的可见性,如果在没有`push`的情况下使用`pop`,编译器将会警告.

## \_\_cplusplus

`__cpluscplus`表示当前的`c++`标准,主要有这几个值

| 值 | 标准 |
| --- | --- |
| `199711L` | `c++98` |
| `201103L` | `c++11` |
| `201402L` | `c++14` |
| `201703L` | `c++17` |
| `202002L` | `c++20` |
| `202100L` | `c++23(不完整)` |

## \_\_attribute\_\_

这个常常这样写

```cpp
__attribute__ ((值))
```

用来提醒编译器,这个函数或数据有一些特殊属性.我整理的如下:

| 值 | 作用 |
| --- | --- |
| `packed` | 取消结构在编译过程中的优化对齐,按照实际占用字节数进行对齐. |
| `aligned(n)` | 指定内存对齐`n`字节 |
| `noreturn` | 告诉编译器这个函数没有返回值,以便编译器优化 |
| `weak` | 弱符号 |
| `__visibility__ (V))` | 可见性,`V`参考上文 |
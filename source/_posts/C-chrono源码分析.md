---
title: C++chrono源码分析
categories:
  - 学习和理论
tags:
  - C++
  - chrono
  - duration
  - ratio
  - time_point
  - 细说std源码
abbrlink: 6ed806ff
date: 2022-02-05 19:58:13
---

* 未写完;

<!-- more -->


## 头文件

```cpp
#include <ratio>
#include <type_traits>
#include <limits>
#include <ctime>
#include <bits/parse_numbers.h> // for literals support.
```

作用如下:

| 头文件 | 作用 |
| --- | --- |
| `ratio` | `ratio`分数类(分子分母是模板参数) |
| `type_traits` | 这里面定义了一些类型判断相关的模板类和模板函数 |
| `limits` | 定义了一些数据类型的最大最小值 |
| `ctime` | `C`语言时间库 |
| `bits/parse_numbers.h` | 用于将字符串转为数值 |

## 超前引用

```cpp
  namespace chrono
  {
    template<typename _Rep, typename _Period = ratio<1>>
      struct duration;

    template<typename _Clock, typename _Dur = typename _Clock::duration>
      struct time_point;
  }
```

这里都参数可以细说.

### ratio

`ratio`定义于`ratio`文件.

```cpp
  template<intmax_t _Num, intmax_t _Den = 1>
    struct ratio
    {
      static_assert(_Den != 0, "denominator cannot be zero");
      static_assert(_Num >= -__INTMAX_MAX__ && _Den >= -__INTMAX_MAX__,
		    "out of range");

      // Note: sign(N) * abs(N) == N
      static constexpr intmax_t num =
        _Num * __static_sign<_Den>::value / __static_gcd<_Num, _Den>::value;

      static constexpr intmax_t den =
        __static_abs<_Den>::value / __static_gcd<_Num, _Den>::value;

      typedef ratio<num, den> type;
    };
```

#### intmax\_t

`intmax_t`在我的电脑上被定义为`long int`,这是平台相关的.

#### 编译期判断

首先判断传入的分母是否为`0`.

```cpp
      static_assert(_Den != 0, "denominator cannot be zero");
```

判断传入的分数是否在范围内

```cpp
      static_assert(_Num >= -__INTMAX_MAX__ && _Den >= -__INTMAX_MAX__,
		    "out of range");
```

#### 编译期处理分子分母

`__static_<XXX>`这些函数的定义如下

```cpp
  template<intmax_t _Pn>
    struct __static_sign
    : integral_constant<intmax_t, (_Pn < 0) ? -1 : 1>
    { };

  template<intmax_t _Pn>
    struct __static_abs
    : integral_constant<intmax_t, _Pn * __static_sign<_Pn>::value>
    { };

  template<intmax_t _Pn, intmax_t _Qn>
    struct __static_gcd
    : __static_gcd<_Qn, (_Pn % _Qn)>
    { };

  template<intmax_t _Pn>
    struct __static_gcd<_Pn, 0>
    : integral_constant<intmax_t, __static_abs<_Pn>::value>
    { };

  template<intmax_t _Qn>
    struct __static_gcd<0, _Qn>
    : integral_constant<intmax_t, __static_abs<_Qn>::value>
    { };
```

我们基本可以把`integral_constant`视为一个在编译期用以表示数值的常量,并且他带有的信息还能用来确定类型.

用法就是像上面那样,`自定义的类型名称 : integral_constant<数值类型, 数值>`.

可以看出:

`__static_sign`根据`_Pn`的值,被定义为`1`或`-1`,来表示符号.

`__static_abs`被定义为`_Pn`乘以`_Pn`的符号,通过负负得正获得绝对值.

`__static_gcd`一路继承下去,最终得到的类是一个`integral_constant`,并且值是`gcd`.

比如,一个`__static_gcd<20,30>`最终会继承一个`integral_constant<intmax_t, 5>`

有人会疑问这里辗转相除次数多了,会不会产生一个很冗余的类.

我这里有一种个人理解:不会.

对于翻译出来的程序来说,`__static_gcd<20,30>`和`integral_constant<intmax_t, 5>`是一样的内容.

如果`A类`仅仅继承`B类`,没有添加其他任何信息,那么这两个类是一样的.没有额外开销.

诸如`std::is_base_of`这样的函数,想要判断类之间的继承关系要怎么判断.

其实编译期直接就把用到继承信息的地方,直接替换成结果,运行期不需要计算,程序本身也不需要得知这些信息.

所以也不需要记录,就不会有直觉上那种冗余信息.

如果带有虚函数,需要在运行器确定类型呢?

那么这个类会有一个虚表,虚表中记录了这些信息,这种情况就另说了.

注意,虚表这个概念只是`C++`实现过程中产生的概念,`C++`应该并未钦定,不要觉得虚表是`C++`的一个天经地义的组成成分.

如果要一个符合普遍规律的说法,应该是
> 内嵌于对象表示中的，位于对象之外的元数据.

```cpp
static constexpr intmax_t num = _Num * __static_sign<_Den>::value / __static_gcd<_Num, _Den>::value;
```

那么,根据上面的代码就是`num`赋值为分子和分母约分之后,带正负的分子.

这个分数的符号由分子确定.

```cpp
static constexpr intmax_t den = __static_abs<_Den>::value / __static_gcd<_Num, _Den>::value;
```

`den`则是分子分母约分之后的分母,并且不带正负.

### \_Clock

内部定义了这几个类,后面会提到.

| 类 | 描述 |
| --- | --- |
| `system_clock` | 从系统获取的时钟 |
| `steady_clock` |不能被修改的时钟 |
| `high_resolution_clock` | 高精度时钟,实际上是`system_clock`或者`steady_clock`的别名 |


## \_\_duration\_common\_type\_wrapper

待续
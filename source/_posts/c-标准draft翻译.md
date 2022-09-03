---
title: c++标准draft翻译
categories:
  - 细说CPP标准和stl源码
tags:
  - C++
  - 语言律师
  - 工地英语
  - 有生之年
mathjax: true
abbrlink: 87a42267
date: 2022-03-05 15:37:28
---

* 将来会翻译完的(心虚);

<!-- more -->

# 0. 前言[todo]

有时候，我们会遇到一些语法不能确定的地方。

这时候我回去翻一下标准。

所以我把我阅读过的部分翻译一下。

随着我越看越多，这里也会越来越多。

目前，文中所见的大部分页内跳转链接都是跳转到文章开头。

# 1. 范围[intro.scope]

1. 这个文档指定了`C++`编程语言的实现要求。本文不仅对`C++`的实现做出了要求，而且也定义了什么是`C++`。这些内容可以在本文的各个位置找到。

2. `C++`是一个基于`ISO/IEC 9899:2018`所描述的C语言的，通用的编程语言。`C++`提供的工具超越了C语言所提供的，包括额外的数据类型、类、模板、异常、命名空间、操作符重载、函数重载、引用、存储释放操作符、和额外的库。

# 2. 参考标准[intro.refs]

1. 本文以某些方式借用了下面的文档中的一些或所有内容。下面列出的文档中：如果标明了日期，那么说明我们只引用了该版本；否则，我们引用的是最新版本。

    1. ISO/IEC 2382, Information technology — Vocabulary
    2. ISO 8601:2004, Data elements and interchange formats — Information interchange — Representation of dates and times
    3. ISO/IEC 9899:2018, Programming languages — C
    4. ISO/IEC/IEEE 9945:2009, Information Technology — Portable Operating System Interface (POSIX)
    5. ISO/IEC/IEEE 9945:2009/Cor 1:2013, Information Technology — Portable Operating System Interface (POSIX), Technical Corrigendum 1
    6. ISO/IEC/IEEE 9945:2009/Cor 2:2017, Information Technology — Portable Operating System Interface (POSIX), Technical Corrigendum 2
    7. ISO/IEC 10646, Information technology — Universal Coded Character Set (UCS)
    8. ISO/IEC 10646:2003, Information technology — Universal Multiple-Octet Coded Character Set (UCS)
    9. ISO/IEC/IEEE 60559:2020, Information technology — Microprocessor Systems — Floating-Point arithmetic
    10. ISO 80000-2:2009, Quantities and units — Part 2: Mathematical signs and symbols to be used in the natural sciences and technology
    11. Ecma International, ECMAScript3 Language Specification, Standard Ecma-262, third edition, 1999.
    12. The Unicode Consortium.
Unicode Standard Annex, UAX #44, Unicode Character Database.
Edited by Ken Whistler and Laurenţiu Iancu.
Available from: [http://www.unicode.org/reports/tr44/]
    13. The Unicode Consortium.
The Unicode Standard, Derived Core Properties.
Available from: [https://www.unicode.org/Public/UCD/latest/ucd/DerivedCoreProperties.txt]

2. `ISO/IEC 9899:2018, Clause 7`所描述的库，在下文被称为`C标准库`。

3. `ISO/IEC 9945:2009`所描述的操作系统接口，在下文被称为`POSIX`。

4. `Ecma-262`所描述的ECMAScript编程语言，在下文被称为`ECMA-262`。

5. （记1：从`ISO/IEC 10646:2003`中引用的内容，只用来支撑一些已经过时特性([depr.locale.stdcvt](#0-前言todo))）。

# 3. 条款和定义[intro.defs]

1. 在`ISO/IEC 2382`中给出的条款和定义，在`ISO 80000-2:2009`中给出的条款、定义、和符号，在本文中依旧适用。

2. ISO和IEC在下面的地址中维护了我们所适用的标准文档。
    1. ISO Online browsing platform: available at [https://www.iso.org/obp]
    2. IEC Electropedia: [available at http://www.electropedia.or]

3. 那些只在本文的一小段中适用的定义，会用斜体来表示。

## 3.1. 访问[defns.access]

（运行时的动作）读取或者修改了对象的值。

（记1：只有标量类型的广义左值可以用来访问对象。标量对象的读取在[conv.lval](#0-前言todo)中有描述，标量对象的修改在[expr.ass](#0-前言todo)、[expr.post.incr](#0-前言todo)和[expr.pre.incr](#0-前言todo)中有描述。尝试读取或者修改对象，将会导致调用类的构造函数或者赋值操作。尽管这些函数可能会执行一些标量子对象的访问操作，但是这个调用本身并不涉及访问。）

## 3.2. 任意位置流[defns.arbitrary.steam]

（库）只要在流的长度范围内，就可以定位到任意整数位置的流。

（记1：所有任意位置流都是可重定位流[defns.prepolitical.stream](#0-前言todo)）

## 3.3. 参数[defns.argument]

（函数调用表达式）在括号中的，被逗号分隔的表达式。

## 3.4. 参数[defns.argument.macro]

（宏函数表达式）在括号中的，被逗号分隔的预处理器标记序列。

## 3.5. 参数[defns.argument.throw]

（抛出异常）[抛出异常](#0-前言todo)的操作数。

## 3.6. 参数[defns.argument.templ]

（模板实例化）在尖括号中的，被逗号分隔的[常量表达式](#0-前言todo)、[类型标识符](#0-前言todo)或者[表达式](#0-前言todo)。

## 3.7. 阻塞[defns.block]

（执行）需要等待一些条件被满足才能继续执行被阻塞的操作。

## 3.8. 代码块[defns.block.stmt]

（语句）一些复合语句。

## 3.9. 字符[defns.character]

（库）被按顺序对待时，可以表示文本的对象。

（记1：不仅仅是意味着`char`、`char_t`、`char16_t`、`char32_t`、`wchar_t`对象，而且包括任何在[字符串](#0-前言todo)、[本地化](#0-前言todo)、[输入输出](#0-前言todo)或[正则](#0-前言todo)中定义的类型）

## 3.10. 字符容器类型[defns.character.container]

（库）用来表示字符的类或者类型

（记1：字符串，输入输出流，正则表达式类模板）

## 3.11. 对照元素[defns.regex.collating.element]

## 3.12. 组件[defns.component]

## 3.13. 条件支持[defns.cond.supp]

## 3.14. 常数子表达式[defns.const.subexpr]

## 3.15. 死锁[defns.deadlock]

## 3.16. 默认行为[defns.default.behavior.impl]

## 3.17. 诊断信息[defns.diagnostic]

## 3.18. [defns.argument]

## 3.19. [defns.argument.macro]

## 3.20. [defns.argument.throw]

## 3.21. [defns.argument.templ]

## 3.22. [defns.block]

## 3.23. [defns.block.stmt]

## 3.24. [defns.character]

## 3.25. 访问[defns.access]

## 3.26. [defns.arbitrary.steam]

## 3.27. [defns.argument]

## 3.28. 访问[defns.access]

## 3.29. [defns.arbitrary.steam]

## 3.30. [defns.argument]

## 3.31. 访问[defns.access]

## 3.32. [defns.arbitrary.steam]

## 3.33. [defns.argument]

## 3.34. [defns.argument.macro]

## 3.35. [defns.argument.throw]

## 3.36. [defns.argument.templ]

## 3.37. [defns.block]

## 3.38. [defns.block.stmt]

## 3.39. [defns.character]

## 3.40. 访问[defns.access]

## 3.41. [defns.arbitrary.steam]

## 3.42. [defns.argument]

## 3.43. 访问[defns.access]

## 3.44. [defns.arbitrary.steam]

## 3.45. [defns.argument]

## 3.46. 访问[defns.access]

## 3.47. [defns.arbitrary.steam]

## 3.48. [defns.argument]

## 3.49. [defns.argument.throw]

## 3.50. [defns.argument.templ]

## 3.51. [defns.block]

## 3.52. [defns.block.stmt]

## 3.53. [defns.character]

## 3.54. 访问[defns.access]

## 3.55. [defns.arbitrary.steam]

## 3.56. [defns.argument]

## 3.57. 访问[defns.access]

## 3.58. [defns.arbitrary.steam]

## 3.59. [defns.argument]

## 3.60. 访问[defns.access]

## 3.61. [defns.arbitrary.steam]

## 3.62. [defns.argument]

## 3.63. 访问[defns.access]

## 3.64. [defns.arbitrary.steam]

## 3.65. [defns.argument]

## 3.66. 访问[defns.access]

## 3.67. [defns.arbitrary.steam]

## 3.68. 良构[defns.well.formed]

结构遵守了语法规则和语义规则的`C++`程序。

# 6. 基本[basic]

## 6.6 程序和链接[basic.link]

1. 一个程序由一个或多个翻译单元链接在一起组成,一个翻译单元由一系列的声明组成.

2. 当一个名字能够表示一个定义在其他作用域的对象/引用/函数/类型/模板/命名空间/变量的时候,他就是一个有链接的名字.

2.1. 当一个名字有外部链接的时候,可以在其他的翻译单元的同作用域,或者同翻译单元的不同作用域中,使用这个名字来引用他表示的实体.

2.2. 当一个名字有模块链接的时候,可以在其他的模块的同作用域,或者同模块的不同作用域中,使用这个名字来引用他表示的实体.

2.3. 当一个名字有内部链接的时候,可以在同一翻译单元的其他作用域中,使用这个名字来引用他表示的实体.

2.4. 当一个名字没有链接时,这个实体不能在其他作用域中,使用名字来引用.

3. 一个属于某个命名空间(全局变量属于全局命名空间)的实体的,如果符合以下任意条件,那么他的名字具有内部链接:

3.1. 显式加上static修饰符的变量,变量模板,函数或函数模板,有内部链接.

3.2. 非volatile,非变量模板的const类型变量有内部链接,除非满足以下任意一点:

3.2.1. 显式声明了extern

3.2.2. 他是inline或exported

3.2.3. 他有前向声明而且他的前向声明没有内部链接

3.3. 匿名联合体的数据成员有内部链接.

## 6.9 程序执行[basic.exec]

### 6.9.3 开始和终止[basic.start]

#### 6.9.3.3 动态初始化和非阻塞变量[basic.start.dynamic]

1. 在静态存储区的非阻塞变量的动态初始化顺序是未被指定的,通常顺序会更加偏向于内联变量.

# 7 表达式[expr]

## 7.6 复合表达式[expr.compound]

### 7.6.1 后缀表达式[expr.post]

#### 7.6.1.5 类成员访问[expr.ref]

# 9 声明[dcl]

## 9.2 标识符[dcl.spec]

### 9.2.9 类型标识符[dcl.type]

#### 9.2.9.6 占位类型标识符[dcl.spec.auto]

##### 9.2.9.6.1 通常[dcl.spec.auto.general]

```
placeholder-type-specifier:
  type-constraint auto
  type-constraint decltype ( auto )
```

1. 占位类型标识符表示一个占位类型,这个占位类型在类型推导之后将会被替换成真正的类型.

## 9.3 声明符[dcl.dcl]

### 9.3.4 声明符的含义[dcl.meaning]

#### 9.3.4.1 通常[dcl.meaning.general]

#### 9.3.4.6 函数[dcl.fct]


# 11 类

## 11.4 类成员

### 11.4.9 静态成员
#### 11.4.9.1 通常

1. 可以使用`X::s`的方式来表示某个类`X`的某个静态成员`s`.不一定需要使用类成员访问[expr.ref]的方式来表示静态成员. 静态成员也可以使用类成员访问的方式来表示,在这种情况下计算对象表达式.

```cpp
struct process {
  static void reschedule();
};
process& g();

void f() {
  process::reschedule();        // OK, no object necessary
  g().reschedule();             // g() is called
```

2. 静态成员遵守对象成员访问规则[class.access].当在类成员的声明中使用的时候,`static`只能用于出现在类定义中的成员声明中.

> 不能用在出现在命名空间的成员声明中.

#### 11.4.9.2 静态成员函数[class.static.mfct]

1. [class.mfct]描述的规则同样适用于静态成员函数.

2. 静态成员函数没有`this`指针[expr.prim.this],静态成员函数不能用`const`, `volatile` 或 `virtual`修饰[dcl.fct]

#### 11.4.9.3 静态数据成员[class.static.data]

1. 静态数据成员不是类的对象的一部分.`thread_local`的静态数据成员会被每个线程拷贝一份.如果静态数据成员没有声明`thread_local`,那么类的所有成员共享着一个数据成员.

2. 静态数据成员不该被`mutable`[dcl.stc].静态数据成员不能是匿名类[class.pre], 局部类[class.local]或内嵌类[class.nest]的直接成员[class.meml].

3. 在类定义中的非内联静态数据成员并未被定义,他可能是除`cv void`以外的未完成类型

> 静态数据成员的初始化位于他所隶属的类的作用域内[basic.scope.class]

```cpp
class process {
  static process* run_chain;
  static process* running;
};

process* process::running = get_main();
process* process::run_chain = running;
```

静态数据成员`run_chain`的定义位于全局作用域;符号`process::run_chain`表示`run_chain`是类`process`的成员,,并且在类`process`的作用域内.

> 一旦静态数据成员被定义,他就会存在,哪怕这个类实例化没有任何对象.

在这个例子,即使类`process`没产生任何对象,`running`和`run_chain`依旧存在.

[basic.start.static], [basic.start.dynameic]和[basic.start.term]描述了静态数据成员的初始化和销毁.

4. 如果一个非内联.非`volatile`的常量静态数据成员是一个整数或枚举类型,在类定义中声明语句中,它可以直接使用一个常量通过赋值表达式来初始化.

5. 在一个合法的程序中一个静态数据成员只能被定义一次.[basic.def.odr]

6. 命名空间访问中的类的静态数据成员具有类名称的链接.[basic.link]

# 12 重载[over]

## 12.2 重载分析[over.match]

### 12.2.4 最可行的函数[over.match.best]

#### 12.2.4.1 通常[over.match.best.general]

#### 12.2.4.2 隐式转换序列[over.best.ics]

##### 12.2.4.2.1 通常[over.best.ics.general]

1. 隐式转换序列,是一用于将函数调用时传入的参数类型转换为被调用的函数所需要的参数类型的.

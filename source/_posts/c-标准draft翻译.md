---
title: c++标准draft翻译
categories:
  - 学习和理论
tags:
  - C++
  - C++语言律师
  - 工地英语
  - 有生之年
mathjax: true
abbrlink: 87a42267
date: 2022-03-05 15:37:28
---

* 将来会翻译完的(心虚);

<!-- more -->

## 前言

有时候,我们会遇到一些语法不能确定的地方.

这时候我回去翻一下标准.

那应该不止我一个人会去标准里找答案,所以我把我阅读过的部分翻译一下.

随着我越看越多,这里也会越来越多.

## 6 基本[basic]

### 6.6 程序和链接[basic.link]

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

### 6.9 程序执行[basic.exec]

#### 6.9.3 开始和终止[basic.start]

##### 6.9.3.3 动态初始化和非阻塞变量[basic.start.dynamic]

1. 在静态存储区的非阻塞变量的动态初始化顺序是未被指定的,通常顺序会更加偏向于内联变量.

## 7 表达式[expr]

## 7.6 复合表达式[expr.compound]

### 7.6.1 后缀表达式[expr.post]

#### 7.6.1.5 类成员访问[expr.ref]

## 9 声明[dcl]

### 9.2 标识符[dcl.spec]

#### 9.2.9 类型标识符[dcl.type]

##### 9.2.9.6 占位类型标识符[dcl.spec.auto]

###### 9.2.9.6.1 通常[dcl.spec.auto.general]

```
placeholder-type-specifier:
  type-constraint auto
  type-constraint decltype ( auto )
```

1. 占位类型标识符表示一个占位类型,这个占位类型在类型推导之后将会被替换成真正的类型.

### 9.3 声明符[dcl.dcl]

#### 9.3.4 声明符的含义[dcl.meaning]

##### 9.3.4.1 通常[dcl.meaning.general]

##### 9.3.4.6 函数[dcl.fct]


## 11 类[class]

### 11.4 类成员[class.mem]

#### 11.4.9 静态成员[class.static]

##### 11.4.9.1 通常[class.static.general]

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

##### 11.4.9.2 静态成员函数[class.static.mfct]

1. [class.mfct]描述的规则同样适用于静态成员函数.

2. 静态成员函数没有`this`指针[expr.prim.this],静态成员函数不能用`const`, `volatile` 或 `virtual`修饰[dcl.fct]

##### 11.4.9.3 静态数据成员[class.static.data]

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

## 12 重载[over]

### 12.2 重载分析[over.match]

#### 12.2.4 最可行的函数[over.match.best]

##### 12.2.4.1 通常[over.match.best.general]

##### 12.2.4.2 隐式转换序列[over.best.ics]

###### 12.2.4.2.1 通常[over.best.ics.general]

1. 隐式转换序列,是一用于将函数调用时传入的参数类型转换为被调用的函数所需要的参数类型的.

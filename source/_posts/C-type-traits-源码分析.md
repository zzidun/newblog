---
title: C++ type traits 源码分析
categories:
  - 学习和理论
tags:
  - C++
  - 模板元编程
  - 细说std源码
abbrlink: ae6bf35b
date: 2022-03-05 14:31:08
---

<!-- more -->

## 宏

```cpp
#ifndef _GLIBCXX_TYPE_TRAITS
#define _GLIBCXX_TYPE_TRAITS 1

#pragma GCC system_header

#if __cplusplus < 201103L
# include <bits/c++0x_warning.h>
#else

#include <bits/c++config.h>
```

首先定义一个宏`_GLIBCXX_TYPE_TRAITS`,以便在重复包含的时候不会出问题.

`#pragma GCC system_header`表示编译器将会像对待系统库那样对待这个文件(也就是这个文件可能不符合标准,但是不要警告).

如果当前版本小于c++11(也就是`__cplusplus < 201103L`),那么`type_traits`只会包含一个`bits/c++0x_warning.h`,这个头文件里有一些警告信息.

`bits/c++config.h`中定义了一些宏,日后细说.

## 命名空间

```cpp
namespace std _GLIBCXX_VISIBILITY(default)
{
_GLIBCXX_BEGIN_NAMESPACE_VERSION
```

这里出现的两个宏的定义位于上面所说的`bits/c++config.h`中.

`_GLIBCXX_VISIBILITY`用于改变命名空间可见性.

如果`_GLIBCXX_INLINE_VERSION`的值不为`0`,`_GLIBCXX_BEGIN_NAMESPACE_VERSION`会在这个位置再定义一个命名空间,用来包住后面的定义.

`_GLIBCXX_INLINE_VERSION`默认是`0`的,这时候`_GLIBCXX_BEGIN_NAMESPACE_VERSION`宏是空的.

## 前向声明

```cpp
  template<typename... _Elements>
    class tuple;

  template<typename _Tp>
    class reference_wrapper;
```

这里前向声明了两个定义在其他头文件里的类,从而不用包含这两个类所在的头文件.

这两个类所在的头文件也包含了`type_traits`,因此`type_traits`不能再去包含他们.

## integral\_constant

```cpp
  template<typename _Tp, _Tp __v>
    struct integral_constant
    {
      static constexpr _Tp                  value = __v;
      typedef _Tp                           value_type;
      typedef integral_constant<_Tp, __v>   type;
      constexpr operator value_type() const noexcept { return value; }
#if __cplusplus > 201103L

#define __cpp_lib_integral_constant_callable 201304

      constexpr value_type operator()() const noexcept { return value; }
#endif
    };
```

### 数据成员

这个结构体定义了变量`value = __v`,这是一个`constexpr static`修饰的数据成员变量.

在C++17之后,这样`constexpr`修饰的静态成员变量可以在类定义语句内初始化.

在C++标准文档中有这样一句话

> A function or static data member declared with the constexpr or consteval specifier is implicitly an inline function or variable
> 被constexpr或consteval修饰的函数或则静态数据成员视为隐式地被inline修饰.

也就是这么做效果等同`inline`.

C++17之后可以给静态数据成员加上`inline`,使得他可以直接在类定义语句内初始化.

接下来是将`_Tp`定义为`value_type`,将自身类型定义为`type`.

### 函数成员

#### operator value_type()

如果将类型`integral_constant<_Tp, __v>`的对象转换为`_Tp`类型,那么将会调用`operator value_type()`函数,返回一个`_Tp`类型的`value`.

如果有类似这样的语句
```cpp
int i = int(integral_constant<int,1>())
```

因为这时候`value_type`为`int`,那么这个类将会有一个函数成员`operator int()`.

例如上面的语句会返回`int`类型的`1`.

#### operator()()

如果有一个`integral_constant<_Tp, __v>`类型的对象`a`,当我们使用`a()`时,将会调用`operator()()`.

这里的`operator()()`直接返回了`value`.

例如,下面的语句,将会返回一个`int`类型的`1`.

```cpp
integral_constant<int, 1>()()
```

### 静态成员定义

```cpp
  template<typename _Tp, _Tp __v>
    constexpr _Tp integral_constant<_Tp, __v>::value;
```

如果C++版本小于17,那么还是需要在类定义语句之外使用这个语句来定义`value`.

在C++标准中有这样一段话

> 在类定义中的非内联静态数据成员并未被定义,他可能是除`cv void`以外的未完成类型

因此我们需要在类的外面定义他.

### 常用类型定义

```cpp
  /// The type used as a compile-time boolean with true value.
  using true_type =  integral_constant<bool, true>;

  /// The type used as a compile-time boolean with false value.
  using false_type = integral_constant<bool, false>;

  /// @cond undocumented
  /// bool_constant for C++11
  template<bool __v>
    using __bool_constant = integral_constant<bool, __v>;
  /// @endcond

#if __cplusplus >= 201703L
# define __cpp_lib_bool_constant 201505
  /// Alias template for compile-time boolean constant types.
  /// @since C++17
  template<bool __v>
    using bool_constant = integral_constant<bool, __v>;
#endif
```

这里使用`using`重命名了几个特定类型的`integral_constant`.

其中`false_type`和`true_type`非常常用,他们是模板元编程的基石之一.

## conditional

这里是一个前向定义.

```cpp
  template<bool, typename, typename>
    struct conditional;
```

他的实现也在同一个文件中.

```cpp
  template<bool _Cond, typename _Iftrue, typename _Iffalse>
    struct conditional
    { typedef _Iftrue type; };

  // Partial specialization for false.
  template<typename _Iftrue, typename _Iffalse>
    struct conditional<false, _Iftrue, _Iffalse>
    { typedef _Iffalse type; };
```

以上代码根据`bool _Cond`的值,来决定`conditional::type`.


首先定义一个默认模板类,无论如何都把`type`定义为`_Iftrue`.

然后特化这个模板,当`_Cond`传入`false`的时候,把`type`定义为`_Iffalse`.

上述的`conditonal`的第一个模板参数`_Cond`其实很有说法.

如果我们直接传布尔值,或者传一个很简单的表达式,那就浪费了这个精巧的结构.

为此,C++定义了`__or_`, `__and_`,`__not_`结构体,来实现复杂的判断.

## \_\_type\_identity

```cpp
  template <typename _Type>
    struct __type_identity
    { using type = _Type; };

  template<typename _Tp>
    using __type_identity_t = typename __type_identity<_Tp>::type;
```

这里定义了一个结构体`__type_identity`,这个结构体内部只是把模板参数中的`_Type`命名为`type`.

然后就是定义一个`__type_identity_t`,以便使用这个`type`.

## 逻辑运算

这几个逻辑运算也是模板元编程中非常关键的一点,他们给模板带来了逻辑.

### \_\_or\_

```cpp
  template<typename...>
    struct __or_;

  template<>
    struct __or_<>
    : public false_type
    { };

  template<typename _B1>
    struct __or_<_B1>
    : public _B1
    { };

  template<typename _B1, typename _B2>
    struct __or_<_B1, _B2>
    : public conditional<_B1::value, _B1, _B2>::type
    { };

  template<typename _B1, typename _B2, typename _B3, typename... _Bn>
    struct __or_<_B1, _B2, _B3, _Bn...>
    : public conditional<_B1::value, _B1, __or_<_B2, _B3, _Bn...>>::type
    { };
```

#### \_\_or\_<\>

当我们不给`__or_`传模板参数时,他就会直接继承一个`false_type`.

这时候这个`__or_`的`value`成员就是`false`.

也就是说以下代码:

```cpp
    std::__or_<> empty_or;
    std::cout << empty_or.value << std::endl;
```

会输出`0`.

#### \_\_or\_<\_B1\>

当`__or_`接受一个模板参数的时候,他直接继承这个这个模板参数.

也就是如果这个参数继承了`true_type`,也就是`_B1::value`为`true`,那么`__or_`就会继承`_B1`,这时候`value`成员就会是`true`.

同理,如果这个参数继承了`false_type`,那么`__or_`就继承`_B1`之后,`value`成员就会是`false`.

这非常符合`or`的特征,当只接收一个布尔值时,计算结果就是这个布尔值.

#### \_\_or\_<\_B1, \_B2\>

如果`_B1::value`为`true`,那么`conditional<_B1::value, _B1, _B2>::type`将会返回`_B1`.

那么`__or_`直接继承`_B1`,短路掉`_B2`.

这时候`value`就是`true.`

否则就继承`_B2`.

继承`_B2`之后,`__or_`的`value`取决于`_B2::value`.

这和`or`运算的表现一模一样.

这就相当于`_B1 || (_B2)`.

#### \_\_or\_<\_B1, \_B2, \_B3, \_Bn...>

这里将传入的模板参数全部展开,就像这样

```cpp
conditional<_B1::value, _B1, __or_<_B2, _B3, _B4>>::type
//展开一次
conditional<_B1::value, _B1, conditional<_B2::value, _B2, __or_<_B3, _B4>>>::type
//再展开一次
conditional<_B1::value, _B1, conditional<_B2::value, _B2, conditional<_B3::type, _B3, _B4>>>::type
```

这就像是`_B1 || (_B2 || (_B3 || _B4))`.


一旦有一个符合,`value`就会是`true`.

### \_\_and\_

```cpp
  template<typename...>
    struct __and_;

  template<>
    struct __and_<>
    : public true_type
    { };

  template<typename _B1>
    struct __and_<_B1>
    : public _B1
    { };

  template<typename _B1, typename _B2>
    struct __and_<_B1, _B2>
    : public conditional<_B1::value, _B2, _B1>::type
    { };

  template<typename _B1, typename _B2, typename _B3, typename... _Bn>
    struct __and_<_B1, _B2, _B3, _Bn...>
    : public conditional<_B1::value, __and_<_B2, _B3, _Bn...>, _B1>::type
    { };
```

不传入模板参数的时候,直接继承`true_type`.

传入一个模板参数的时候,也是直接继承`_B1`.

传入两个模板参数的时候,如果`_B1`为`false`,那么直接继承`_B1`; 否则继承`_B2`,这时候`__and_`的值取决于`_B2`.

后面的展开也和上面同理,相当于展开成`_B1 && (_B2 && (_B3 && ...))`.

### \_\_not\_

```cpp
  template<typename _Pp>
    struct __not_
    : public __bool_constant<!bool(_Pp::value)>
    { };
```

这里就简单很多直接对传入模板参数的`value`取反,然后继承一个`__bool_constant`.

### 快捷方式

```cpp
#if __cplusplus >= 201703L

  /// @cond undocumented
  template<typename... _Bn>
    inline constexpr bool __or_v = __or_<_Bn...>::value;
  template<typename... _Bn>
    inline constexpr bool __and_v = __and_<_Bn...>::value;

	#define __cpp_lib_logical_traits 201510

  template<typename... _Bn>
    struct conjunction
    : __and_<_Bn...>
    { };

  template<typename... _Bn>
    struct disjunction
    : __or_<_Bn...>
    { };

  template<typename _Pp>
    struct negation
    : __not_<_Pp>
    { };

	template<typename... _Bn>
    inline constexpr bool conjunction_v = conjunction<_Bn...>::value;

  template<typename... _Bn>
    inline constexpr bool disjunction_v = disjunction<_Bn...>::value;

  template<typename _Pp>
    inline constexpr bool negation_v = negation<_Pp>::value;

#endif
```

这里定义了`inline constexpr`的变量`__or_v`和`__and_v`,可以方便地访问`value`.

目前`constexpr`模板变量直接写在头文件里没有关系,因为他没有外部链接,但是以后可能会有问题.

加上`inline`的变量可以定义在头文件里,然后被其他文件包含也不会出问题.

## 类型判断

类型判断的模板类我打乱顺序来讲,但是这样更加易于理解.

### is\_reference

```cpp
	template<typename>
    struct is_lvalue_reference
    : public false_type { };

  template<typename _Tp>
    struct is_lvalue_reference<_Tp&>
    : public true_type { };

  template<typename>
    struct is_rvalue_reference
    : public false_type { };

  template<typename _Tp>
    struct is_rvalue_reference<_Tp&&>
    : public true_type { };

  template<typename _Tp>
    struct is_reference
    : public __or_<is_lvalue_reference<_Tp>,
                   is_rvalue_reference<_Tp>>::type
    { };
```

这里用来判断`_Tp`是否引用类型,如果了解模板的匹配就会知道,如果是左值或则右值引用,就会继承`true_type`.

不必多说,很简单.

### is\_const

```cpp
  /// is_const
  template<typename>
    struct is_const
    : public false_type { };

  template<typename _Tp>
    struct is_const<_Tp const>
    : public true_type { };
```

这也同理,如果带有`const`就会选择第二个特化.

### is\_function

```cpp
  template<typename _Tp>
    struct is_function
    : public __bool_constant<!is_const<const _Tp>::value> { };

  template<typename _Tp>
    struct is_function<_Tp&>
    : public false_type { };

  template<typename _Tp>
    struct is_function<_Tp&&>
    : public false_type { };
```

如果在`_Tp`的前面加上`const`之后,`_Tp`的类型不为`const`,那么这个类型就是一个函数.

这里非常巧妙.

我们知道,在函数前面加上`const`,他只是变成了一个返回`const`类型的函数.

### is\_void

```cpp

  template<typename _Tp>
    using __remove_cv_t = typename remove_cv<_Tp>::type;

  template<typename>
    struct __is_void_helper
    : public false_type { };

  template<>
    struct __is_void_helper<void>
    : public true_type { };

  template<typename _Tp>
    struct is_void
    : public __is_void_helper<__remove_cv_t<_Tp>>::type
    { };
```

`__remove_cv_t`作用是将`_Tp`的`const`和`volatile`修饰符去掉(如果有的话).后面会提到他的实现

然后直接和`void`类型对比,作出判断.

## 类型操作

### 修改const和volatile

```cpp
  // Const-volatile modifications.

  /// remove_const
  template<typename _Tp>
    struct remove_const
    { typedef _Tp     type; };

  template<typename _Tp>
    struct remove_const<_Tp const>
    { typedef _Tp     type; };

  /// remove_volatile
  template<typename _Tp>
    struct remove_volatile
    { typedef _Tp     type; };

  template<typename _Tp>
    struct remove_volatile<_Tp volatile>
    { typedef _Tp     type; };

  /// remove_cv
  template<typename _Tp>
    struct remove_cv
    { using type = _Tp; };

  template<typename _Tp>
    struct remove_cv<const _Tp>
    { using type = _Tp; };

  template<typename _Tp>
    struct remove_cv<volatile _Tp>
    { using type = _Tp; };

  template<typename _Tp>
    struct remove_cv<const volatile _Tp>
    { using type = _Tp; };

  /// add_const
  template<typename _Tp>
    struct add_const
    { typedef _Tp const     type; };

  /// add_volatile
  template<typename _Tp>
    struct add_volatile
    { typedef _Tp volatile     type; };

  /// add_cv
  template<typename _Tp>
    struct add_cv
    {
      typedef typename
      add_const<typename add_volatile<_Tp>::type>::type     type;
    };

#if __cplusplus > 201103L

#define __cpp_lib_transformation_trait_aliases 201304

  /// Alias template for remove_const
  template<typename _Tp>
    using remove_const_t = typename remove_const<_Tp>::type;

  /// Alias template for remove_volatile
  template<typename _Tp>
    using remove_volatile_t = typename remove_volatile<_Tp>::type;

  /// Alias template for remove_cv
  template<typename _Tp>
    using remove_cv_t = typename remove_cv<_Tp>::type;

  /// Alias template for add_const
  template<typename _Tp>
    using add_const_t = typename add_const<_Tp>::type;

  /// Alias template for add_volatile
  template<typename _Tp>
    using add_volatile_t = typename add_volatile<_Tp>::type;

  /// Alias template for add_cv
  template<typename _Tp>
    using add_cv_t = typename add_cv<_Tp>::type;
#endif
```

操作非常简单,接收特定类型之后在模板类内部定义一个自己想用的版本就可以了.

### 修改引用类型

```cpp
  /// remove_reference
  template<typename _Tp>
    struct remove_reference
    { typedef _Tp   type; };

  template<typename _Tp>
    struct remove_reference<_Tp&>
    { typedef _Tp   type; };

  template<typename _Tp>
    struct remove_reference<_Tp&&>
    { typedef _Tp   type; };

  template<typename _Tp, bool = __is_referenceable<_Tp>::value>
    struct __add_lvalue_reference_helper
    { typedef _Tp   type; };

  template<typename _Tp>
    struct __add_lvalue_reference_helper<_Tp, true>
    { typedef _Tp&   type; };

  /// add_lvalue_reference
  template<typename _Tp>
    struct add_lvalue_reference
    : public __add_lvalue_reference_helper<_Tp>
    { };

  template<typename _Tp, bool = __is_referenceable<_Tp>::value>
    struct __add_rvalue_reference_helper
    { typedef _Tp   type; };

  template<typename _Tp>
    struct __add_rvalue_reference_helper<_Tp, true>
    { typedef _Tp&&   type; };

  /// add_rvalue_reference
  template<typename _Tp>
    struct add_rvalue_reference
    : public __add_rvalue_reference_helper<_Tp>
    { };

#if __cplusplus > 201103L
  /// Alias template for remove_reference
  template<typename _Tp>
    using remove_reference_t = typename remove_reference<_Tp>::type;

  /// Alias template for add_lvalue_reference
  template<typename _Tp>
    using add_lvalue_reference_t = typename add_lvalue_reference<_Tp>::type;

  /// Alias template for add_rvalue_reference
  template<typename _Tp>
    using add_rvalue_reference_t = typename add_rvalue_reference<_Tp>::type;
#endif
```

也是一样.

## 更多类型判断

##
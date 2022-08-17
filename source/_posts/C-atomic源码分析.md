---
title: C++atomic源码分析
categories:
  - 学习和理论
tags:
  - C++
  - atomic
  - 细说std源码
abbrlink: 51d9423d
date: 2022-02-06 13:12:33
---

* 未写完;

<!-- more -->

## atomic的那些特化

`atomic`主要有4种特化:

1. 通用实现;

2. 针对`bool`类型的特化;

3. 针对指针类型的特化;

4. 针对普通数值类型的特化;

## 针对普通数值类型的特化

对于以下的类型,统统是继承了`__atomic_base<type>`.

1. `char; signed char; unsigned char;`
2. `short; unsigned short;`
3. `int; unsigned int;`
4. `long; unsigned long;`
5. `long long; unsigned long long;`
6. `wchar_t;`
7. `char16_t; char32_t;`

他们的特化实现都一样,形如:

```cpp
  template<>
    struct atomic<char> : __atomic_base<char>
    {
      typedef char 			__integral_type;
      typedef __atomic_base<char> 	__base_type;

      atomic() noexcept = default;
      ~atomic() noexcept = default;
      atomic(const atomic&) = delete;
      atomic& operator=(const atomic&) = delete;
      atomic& operator=(const atomic&) volatile = delete;

      constexpr atomic(__integral_type __i) noexcept : __base_type(__i) { }

      using __base_type::operator __integral_type;
      using __base_type::operator=;

#if __cplusplus >= 201703L
    static constexpr bool is_always_lock_free = ATOMIC_CHAR_LOCK_FREE == 2;
#endif
    };
```

### 构造方法

`atomic()`和`~atomioc()`被定义为`default`.

`atomic(char)`则是直接初始化给私有成员`__i`.

其他都`delete`了.

我暂且不知道为什么不允许拷贝`atomic`.

### volatile赋值操作符

对于`volatile`修饰的对象,赋值时会调用`volatile`修饰的赋值操作符.

### delete和using

可以看到,这里`delete`了`operator=`,但是却`using`了父类的`operator=`.

这样做,只是禁用了`atomic& operator=(const atomic&)`这个版本的`operator`.

但是父类的`operator=()`定义了参数为`char`时的行为,则不会被`delete`.

也就是这样:

```cpp
std::atomic<char> x;
std::atomic<char> y;
x = y; // 语法错误
x = (char)1; // 可以
```

### atomic\_base

除了以上内容,atomic继承了`__atomic_base`.我们来细说他.

#### 一些类型定义

```cpp
  template<typename _ITp>
    struct __atomic_base
    {
      using value_type = _ITp;
      using difference_type = value_type;

    private:
      typedef _ITp 	__int_type;

      static constexpr int _S_alignment =
	sizeof(_ITp) > alignof(_ITp) ? sizeof(_ITp) : alignof(_ITp);

      alignas(_S_alignment) __int_type _M_i;
```

这里把传入的数值类型命名为`value_type`.

这里的`difference_type`是用于描述差值的类型,显然数值和数值的差值,用的还是这个数值本身的类型.

`_S_alignment`取`_ITp`的大小和对齐尺寸中的最大值.

众所周知,数值类型的大小和对齐方式也许不一样,比如在`32`位系统中,`long double`大小为`12`字节,但是却按照`4`字节对齐.

但是注意,这里没有特化`long double`.也就是所有情况下应该都相同,并且都是`2`的次幂.

`alignas(_S_alignment) __int_type _M_i;`,声明一个按照`_S_alignment`对齐的,`_ITp`类型的成员.

这个成员就是原子类要操作的那个数值了.

#### 构造方法

```cpp
    public:
      __atomic_base() noexcept = default;
      ~__atomic_base() noexcept = default;
      __atomic_base(const __atomic_base&) = delete;
      __atomic_base& operator=(const __atomic_base&) = delete;
      __atomic_base& operator=(const __atomic_base&) volatile = delete;

      // Requires __int_type convertible to _M_i.
      constexpr __atomic_base(__int_type __i) noexcept : _M_i (__i) { }
```

同样禁用了拷贝构造,而且可以直接从一个数值初始化.


#### 操作数值

```cpp
      operator __int_type() const noexcept
      { return load(); }

      operator __int_type() const volatile noexcept
      { return load(); }

      __int_type
      operator=(__int_type __i) noexcept
      {
	store(__i);
	return __i;
      }

      __int_type
      operator=(__int_type __i) volatile noexcept
      {
	store(__i);
	return __i;
      }
```

当我们把一个`atomic<char>`类型强转为`char`时,就会调用`operator __int_type()`.从而取出数值.

而第二个函数就是上面所说的,子类`using::operator=(char)`的函数.

```cpp
      __int_type
      operator++(int) noexcept
      { return fetch_add(1); }

      __int_type
      operator++(int) volatile noexcept
      { return fetch_add(1); }

      __int_type
      operator--(int) noexcept
      { return fetch_sub(1); }

      __int_type
      operator--(int) volatile noexcept
      { return fetch_sub(1); }

      __int_type
      operator++() noexcept
      { return __atomic_add_fetch(&_M_i, 1, int(memory_order_seq_cst)); }

      __int_type
      operator++() volatile noexcept
      { return __atomic_add_fetch(&_M_i, 1, int(memory_order_seq_cst)); }

      __int_type
      operator--() noexcept
      { return __atomic_sub_fetch(&_M_i, 1, int(memory_order_seq_cst)); }

      __int_type
      operator--() volatile noexcept
      { return __atomic_sub_fetch(&_M_i, 1, int(memory_order_seq_cst)); }

      __int_type
      operator+=(__int_type __i) noexcept
      { return __atomic_add_fetch(&_M_i, __i, int(memory_order_seq_cst)); }

      __int_type
      operator+=(__int_type __i) volatile noexcept
      { return __atomic_add_fetch(&_M_i, __i, int(memory_order_seq_cst)); }

      __int_type
      operator-=(__int_type __i) noexcept
      { return __atomic_sub_fetch(&_M_i, __i, int(memory_order_seq_cst)); }

      __int_type
      operator-=(__int_type __i) volatile noexcept
      { return __atomic_sub_fetch(&_M_i, __i, int(memory_order_seq_cst)); }

      __int_type
      operator&=(__int_type __i) noexcept
      { return __atomic_and_fetch(&_M_i, __i, int(memory_order_seq_cst)); }

      __int_type
      operator&=(__int_type __i) volatile noexcept
      { return __atomic_and_fetch(&_M_i, __i, int(memory_order_seq_cst)); }

      __int_type
      operator|=(__int_type __i) noexcept
      { return __atomic_or_fetch(&_M_i, __i, int(memory_order_seq_cst)); }

      __int_type
      operator|=(__int_type __i) volatile noexcept
      { return __atomic_or_fetch(&_M_i, __i, int(memory_order_seq_cst)); }

      __int_type
      operator^=(__int_type __i) noexcept
      { return __atomic_xor_fetch(&_M_i, __i, int(memory_order_seq_cst)); }

      __int_type
      operator^=(__int_type __i) volatile noexcept
      { return __atomic_xor_fetch(&_M_i, __i, int(memory_order_seq_cst)); }
```

这里逻辑非常简单,我们暂时认为他们调用的函数是原子性操作.

整理用到的函数如下,我们后面会分析:

| 函数 | 相当于原子性的 |
| --- | --- |
| `load` | 读 |
| `store` | 写 |
| `fetch_add` | `x++` |
| `fetch_sub` | `x--` |
| `__atomic_add_fetch` | `x+=i` |
| `__atomic_sub_fetch` | `x-=i` |
| `__atomic_and_fetch` | `x&=i` |
| `__atomic_or_fetch` | `x|=i` |
| `__atomic_xor_fetch` | `x^=i` |

这里要注意一目运算符重载的方法,我们可以认为:

`operator++(int)`是重载了`i++`操作.

这里的`int`参数纯粹用于将它和`++i`区别开来,不会去使用这个参数.

`operator++()`是重载了`++i`操作.

其他同理.


#### 检查这个对象的原子操作是否免锁

```cpp

      bool
      is_lock_free() const noexcept
      {
	// Use a fake, minimally aligned pointer.
	return __atomic_is_lock_free(sizeof(_M_i),
	    reinterpret_cast<void *>(-_S_alignment));
      }

      bool
      is_lock_free() const volatile noexcept
      {
	// Use a fake, minimally aligned pointer.
	return __atomic_is_lock_free(sizeof(_M_i),
	    reinterpret_cast<void *>(-_S_alignment));
      }
```

`reinterpret_cast<void *>`用`void*`的方式来理解`-_S_alignment`的每一个比特位.

也就是把`-_S_alignment`当成一个`void*`指针.

不管`-_S_alignment`原本是什么含义,而只看他字面上的内容.

我们无法找到`__atomic_is_lock_free`这个函数,因为他是被重命名而来的.

观察以下这个宏

```cpp

#define C2_(X,Y)        X ## Y
#define C2(X,Y)                C2_(X,Y)

#define S2(X)                #X
#define S(X)                S2(X)

#define ASMNAME(X)        __asm__(S(C2(__USER_LABEL_PREFIX__,X)))

# define EXPORT_ALIAS(X)                                        \
        extern typeof(C2(libat_,X)) C2(export_,X)                \
          ASMNAME(C2(__atomic_,X))                                \
          __attribute__((alias(S(C2(libat_,X)))))

```

这里用一个宏把`libat_<X>`重命名为`__atomic_<X>`.

`#X`表示把`X`变为字符串,这里用到两次.

用于拼接产生一个字符串,作为参数传给`__asm__`和`alias`.

`X ## Y`用于拼接两个字符串,在这里基本就是用来拼接函数名.

`alias`重命名函数,用法像是这样:

```cpp
typeof(oldname) newname __attribute__((alias("oldname")));
```

`__USER_LABEL_PREFIX__`见这个[链接](https://gcc.gnu.org/onlinedocs/cpp/Common-Predefined-Macros.html),他看起来是空的.

总之,`EXPORT_ALIAS`将会翻译为:

```cpp
typeof(libat_X) export_X __asm__("__atomic_X") __attribute__((alias("libat_X")));
```

这里是给他起了两个名字,`export_X`和`__atomic_X`.

如果你想要自己试一下这个语法,编译时会发现找不到符号`libat_X`.

因为`C++`的符号名不是函数名,你用`gcc`编译就可以了.

这个`__atomic_is_lock_free`的实际实现如下:

```cpp
#define EXACT(N)                                                \
  do {                                                                \
    if (!C2(HAVE_INT,N)) break; /*检查是否有长度为N的整数,检查宏HAVE_INTN即可判断*/ \
    if ((uintptr_t)ptr & (N - 1)) break; /*检查是否按照N(2的次幂)字节对齐*/ \
    if (__atomic_always_lock_free(N, 0)) return true; \
    if (!C2(MAYBE_HAVE_ATOMIC_CAS_,N)) break; \
    if (C2(FAST_ATOMIC_LDST_,N)) return true; \
  } while (0)

#define LARGER(N)                                                \
  do {                                                                \
    uintptr_t r = (uintptr_t)ptr & (N - 1);                        \
    if (!C2(HAVE_INT,N)) break;                                        \
    if (!C2(FAST_ATOMIC_LDST_,N)) break;                        \
    if (!C2(MAYBE_HAVE_ATOMIC_CAS_,N)) break;                        \
    if (r + n <= N) return true;                                \
  } while (0)

bool
libat_is_lock_free (size_t n, void *ptr)
{
  switch (n)
    {
    case 0:                                return true;
    case 1:                EXACT(1);        goto L4;
    case 2:                EXACT(2);        goto L4;
    case 4:                EXACT(4);        goto L8;
    case 8:                EXACT(8);        goto L16;
    case 16:                EXACT(16);        break;
    case 3: L4:                LARGER(4);        /* FALLTHRU */
    case 5 ... 7: L8:        LARGER(8);        /* FALLTHRU */
    case 9 ... 15: L16:        LARGER(16);        break;
    }
  return false;
}
EXPORT_ALIAS (is_lock_free);
```

总之就是对很多宏进行判断,可以看出,如果类型是一个长度为`N`并且按照`N`字节对齐的数值.

那么就会进入下面的函数判断.其他宏我将会整理.

```cpp
#define ATOMIC_ALWAYS_LOCK_FREE_OR_ALIGNED_LOCK_FREE(size, p)                  \
  (__atomic_always_lock_free(size, p) ||                                       \
   (__atomic_always_lock_free(size, 0) && ((uintptr_t)p % size) == 0))
#define IS_LOCK_FREE_1(p) ATOMIC_ALWAYS_LOCK_FREE_OR_ALIGNED_LOCK_FREE(1, p)
#define IS_LOCK_FREE_2(p) ATOMIC_ALWAYS_LOCK_FREE_OR_ALIGNED_LOCK_FREE(2, p)
#define IS_LOCK_FREE_4(p) ATOMIC_ALWAYS_LOCK_FREE_OR_ALIGNED_LOCK_FREE(4, p)
#define IS_LOCK_FREE_8(p) ATOMIC_ALWAYS_LOCK_FREE_OR_ALIGNED_LOCK_FREE(8, p)
#define IS_LOCK_FREE_16(p) ATOMIC_ALWAYS_LOCK_FREE_OR_ALIGNED_LOCK_FREE(16, p)

#define TRY_LOCK_FREE_CASE(n, type, ptr)                                       \
  case n:                                                                      \
    if (IS_LOCK_FREE_##n(ptr)) {                                               \
      LOCK_FREE_ACTION(type);                                                  \
    }                                                                          \
    break;

#define LOCK_FREE_CASES(ptr)                                                   \
  do {                                                                         \
    switch (size) {                                                            \
      TRY_LOCK_FREE_CASE(1, uint8_t, ptr)                                      \
      TRY_LOCK_FREE_CASE(2, uint16_t, ptr)                                     \
      TRY_LOCK_FREE_CASE(4, uint32_t, ptr)                                     \
      TRY_LOCK_FREE_CASE(8, uint64_t, ptr)                                     \
      TRY_LOCK_FREE_CASE_16(ptr) /* __uint128_t may not be supported */        \
    default:                                                                   \
      break;                                                                   \
    }                                                                          \
  } while (0)

bool __atomic_is_lock_free_c(size_t size, void *ptr) {
#define LOCK_FREE_ACTION(type) return true;
  LOCK_FREE_CASES(ptr);
#undef LOCK_FREE_ACTION
  return false;
```

函数开始定义一个宏`LOCK_FREE_ACTION(type)`,用于返回`true`.


可以看出,他把指针和大小一路传到`__atomic_always_lock_free`,我实在早不到实现,但是在[这个地方](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)找到了他的说明.

> 如果大小字节的对象始终为目标架构生成无锁定原子指令，则此内置函数返回true。大小必须解析为编译时常数，结果也可以解析为编译时常数。
> PTR是可选指向对象的可选指针，可用于确定对齐。值为0表示应使用典型的对齐。编译器也可能忽略此参数。

如果`(__atomic_always_lock_free(size, p)`返回真,或`__atomic_always_lock_free(size, 0)`返回真而且指针是`size`的倍数,那么就是无锁的.

否则将继续检查接下来的宏,直到返回.

#### memory_order



#### load和store操作

```cpp
      _GLIBCXX_ALWAYS_INLINE void
      store(__int_type __i, memory_order __m = memory_order_seq_cst) noexcept
      {
	memory_order __b = __m & __memory_order_mask;
	__glibcxx_assert(__b != memory_order_acquire);
	__glibcxx_assert(__b != memory_order_acq_rel);
	__glibcxx_assert(__b != memory_order_consume);

	__atomic_store_n(&_M_i, __i, int(__m));
      }

      _GLIBCXX_ALWAYS_INLINE void
      store(__int_type __i,
	    memory_order __m = memory_order_seq_cst) volatile noexcept
      {
	memory_order __b = __m & __memory_order_mask;
	__glibcxx_assert(__b != memory_order_acquire);
	__glibcxx_assert(__b != memory_order_acq_rel);
	__glibcxx_assert(__b != memory_order_consume);

	__atomic_store_n(&_M_i, __i, int(__m));
      }

      _GLIBCXX_ALWAYS_INLINE __int_type
      load(memory_order __m = memory_order_seq_cst) const noexcept
      {
	memory_order __b = __m & __memory_order_mask;
	__glibcxx_assert(__b != memory_order_release);
	__glibcxx_assert(__b != memory_order_acq_rel);

	return __atomic_load_n(&_M_i, int(__m));
      }

      _GLIBCXX_ALWAYS_INLINE __int_type
      load(memory_order __m = memory_order_seq_cst) const volatile noexcept
      {
	memory_order __b = __m & __memory_order_mask;
	__glibcxx_assert(__b != memory_order_release);
	__glibcxx_assert(__b != memory_order_acq_rel);

	return __atomic_load_n(&_M_i, int(__m));
      }

      _GLIBCXX_ALWAYS_INLINE __int_type
      exchange(__int_type __i,
	       memory_order __m = memory_order_seq_cst) noexcept
      {
	return __atomic_exchange_n(&_M_i, __i, int(__m));
      }


      _GLIBCXX_ALWAYS_INLINE __int_type
      exchange(__int_type __i,
	       memory_order __m = memory_order_seq_cst) volatile noexcept
      {
	return __atomic_exchange_n(&_M_i, __i, int(__m));
      }

      _GLIBCXX_ALWAYS_INLINE bool
      compare_exchange_weak(__int_type& __i1, __int_type __i2,
			    memory_order __m1, memory_order __m2) noexcept
      {
	memory_order __b2 = __m2 & __memory_order_mask;
	memory_order __b1 = __m1 & __memory_order_mask;
	__glibcxx_assert(__b2 != memory_order_release);
	__glibcxx_assert(__b2 != memory_order_acq_rel);
	__glibcxx_assert(__b2 <= __b1);

	return __atomic_compare_exchange_n(&_M_i, &__i1, __i2, 1,
					   int(__m1), int(__m2));
      }

      _GLIBCXX_ALWAYS_INLINE bool
      compare_exchange_weak(__int_type& __i1, __int_type __i2,
			    memory_order __m1,
			    memory_order __m2) volatile noexcept
      {
	memory_order __b2 = __m2 & __memory_order_mask;
	memory_order __b1 = __m1 & __memory_order_mask;
	__glibcxx_assert(__b2 != memory_order_release);
	__glibcxx_assert(__b2 != memory_order_acq_rel);
	__glibcxx_assert(__b2 <= __b1);

	return __atomic_compare_exchange_n(&_M_i, &__i1, __i2, 1,
					   int(__m1), int(__m2));
      }

      _GLIBCXX_ALWAYS_INLINE bool
      compare_exchange_weak(__int_type& __i1, __int_type __i2,
			    memory_order __m = memory_order_seq_cst) noexcept
      {
	return compare_exchange_weak(__i1, __i2, __m,
				     __cmpexch_failure_order(__m));
      }

      _GLIBCXX_ALWAYS_INLINE bool
      compare_exchange_weak(__int_type& __i1, __int_type __i2,
		   memory_order __m = memory_order_seq_cst) volatile noexcept
      {
	return compare_exchange_weak(__i1, __i2, __m,
				     __cmpexch_failure_order(__m));
      }

      _GLIBCXX_ALWAYS_INLINE bool
      compare_exchange_strong(__int_type& __i1, __int_type __i2,
			      memory_order __m1, memory_order __m2) noexcept
      {
	memory_order __b2 = __m2 & __memory_order_mask;
	memory_order __b1 = __m1 & __memory_order_mask;
	__glibcxx_assert(__b2 != memory_order_release);
	__glibcxx_assert(__b2 != memory_order_acq_rel);
	__glibcxx_assert(__b2 <= __b1);

	return __atomic_compare_exchange_n(&_M_i, &__i1, __i2, 0,
					   int(__m1), int(__m2));
      }

      _GLIBCXX_ALWAYS_INLINE bool
      compare_exchange_strong(__int_type& __i1, __int_type __i2,
			      memory_order __m1,
			      memory_order __m2) volatile noexcept
      {
	memory_order __b2 = __m2 & __memory_order_mask;
	memory_order __b1 = __m1 & __memory_order_mask;

	__glibcxx_assert(__b2 != memory_order_release);
	__glibcxx_assert(__b2 != memory_order_acq_rel);
	__glibcxx_assert(__b2 <= __b1);

	return __atomic_compare_exchange_n(&_M_i, &__i1, __i2, 0,
					   int(__m1), int(__m2));
      }

      _GLIBCXX_ALWAYS_INLINE bool
      compare_exchange_strong(__int_type& __i1, __int_type __i2,
			      memory_order __m = memory_order_seq_cst) noexcept
      {
	return compare_exchange_strong(__i1, __i2, __m,
				       __cmpexch_failure_order(__m));
      }

      _GLIBCXX_ALWAYS_INLINE bool
      compare_exchange_strong(__int_type& __i1, __int_type __i2,
		 memory_order __m = memory_order_seq_cst) volatile noexcept
      {
	return compare_exchange_strong(__i1, __i2, __m,
				       __cmpexch_failure_order(__m));
      }
```

#### 原子其他操作


```cpp
      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_add(__int_type __i,
		memory_order __m = memory_order_seq_cst) noexcept
      { return __atomic_fetch_add(&_M_i, __i, int(__m)); }

      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_add(__int_type __i,
		memory_order __m = memory_order_seq_cst) volatile noexcept
      { return __atomic_fetch_add(&_M_i, __i, int(__m)); }

      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_sub(__int_type __i,
		memory_order __m = memory_order_seq_cst) noexcept
      { return __atomic_fetch_sub(&_M_i, __i, int(__m)); }

      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_sub(__int_type __i,
		memory_order __m = memory_order_seq_cst) volatile noexcept
      { return __atomic_fetch_sub(&_M_i, __i, int(__m)); }

      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_and(__int_type __i,
		memory_order __m = memory_order_seq_cst) noexcept
      { return __atomic_fetch_and(&_M_i, __i, int(__m)); }

      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_and(__int_type __i,
		memory_order __m = memory_order_seq_cst) volatile noexcept
      { return __atomic_fetch_and(&_M_i, __i, int(__m)); }

      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_or(__int_type __i,
	       memory_order __m = memory_order_seq_cst) noexcept
      { return __atomic_fetch_or(&_M_i, __i, int(__m)); }

      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_or(__int_type __i,
	       memory_order __m = memory_order_seq_cst) volatile noexcept
      { return __atomic_fetch_or(&_M_i, __i, int(__m)); }

      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_xor(__int_type __i,
		memory_order __m = memory_order_seq_cst) noexcept
      { return __atomic_fetch_xor(&_M_i, __i, int(__m)); }

      _GLIBCXX_ALWAYS_INLINE __int_type
      fetch_xor(__int_type __i,
		memory_order __m = memory_order_seq_cst) volatile noexcept
      { return __atomic_fetch_xor(&_M_i, __i, int(__m)); }
    };
```

---
title: C++thread源码分析
categories:
  - 学习和理论
tags:
  - C++
  - 线程
  - 细说std源码
abbrlink: 770726c0
date: 2022-01-31 21:43:54
---

* thread的内部类;
* thread的构造函数;
* 执行流程;
* 一些对外函数的实现;

<!-- more -->

## 引言

如果写一段这样的代码:

```cpp
#include <bits/stdc++.h>

struct node {

    node() { printf("%p, cons\n", this); }
    node(node &n) { printf("%p, copy from %p\n", this, &n); }
    node(node &&n) { printf("%p, move from %p\n", this, &n); }
    ~node() { printf("%p,dest\n", this); }
};

void fun(struct node nd)
{
    std::this_thread::sleep_for(std::chrono::seconds(5));
    printf("%p\n", &nd);
}

int main()
{
    printf("1\n");
    std::thread t(fun, node());
    printf("2\n");
    t.join();
    printf("3\n");
    return 0;
}
```

编译命令为:
```shell
g++ a.cpp -lpthread
```

输出结果将会类似这样:

```c++
1
0x7fff2e724a3f, cons
0x7fff2e7249f8, move from 0x7fff2e724a3f
0x5589443292c8, move from 0x7fff2e7249f8
0x7fff2e7249f8,dest
0x7fff2e724a3f,dest
2
0x7fd6a8499df7, move from 0x5589443292c8
0x7fd6a8499df7
0x7fd6a8499df7,dest
0x5589443292c8,dest
3
```

可以看出,一开始构造了一个对象位于栈内存`0x7fff2e724a3f`.

后来他被移动构造到栈内存`0x7fff2e7249f8`,

再后来他被移动到堆内存`0x5589443292c8`,

然后按照先构造后析构的顺序,析构了`0x7fff2e7249f8`, `0x7fff2e724a3f`.

线程执行之后,他被移动到栈内存`0x7fd6a8499df7`,等待五秒并打印了这个地址.

函数执行结束,析构了`0x7fd6a8499df7`.

在线程结束时,析构了`0x5589443292c8`.

看完下面文章,你会清楚这些构造,移动和析构,都在什么函数,什么时机中执行.


## 文件开头定义

### 宏

```cpp
#ifndef _GLIBCXX_THREAD
#define _GLIBCXX_THREAD 1
```

定义`_GLIBCXX_THREAD`宏.


```cpp
#pragma GCC system_header
```

从`#pragma GCC system_header`直到文件结束之间的代码会被编译器视为系统头文件之中的代码。系统头文件中的代码往往不能完全遵循C标准, 所以头文件之中的警告信息往往不显示。(除非用`#warning`显式指明)。

```cpp
#if __cplusplus < 201103L
# include <bits/c++0x_warning.h>
#else
```

`__cpluscplus`表示当前的`c++`标准,主要有这几个值

| 值 | 标准 |
| --- | --- |
| `199711L` | `c++98` |
| `201103L` | `c++11` |
| `201402L` | `c++14` |
| `201703L` | `c++17` |
| `202002L` | `c++20` |
| `202100L` | `c++23(不完整)` |

显然他们是单调递增的,`c++11`之前的`__cpluscplus`都会小于`201103L`.

这里表示如果当前标准小于`c++11`,那么不引入下面的代码,而是直接引入`<bits/c++0x_warning.h>`.

### 头文件

```cpp
#include <chrono>
#include <memory>
#include <tuple>
#include <cerrno>
#include <bits/functexcept.h>
#include <bits/functional_hash.h>
#include <bits/invoke.h>
#include <bits/gthr.h>
```

作用如下:

| 头文件 | 作用 |
| --- | --- |
| `chrono` | 时间日期相关 |
| `memory` | 内存分配,回收和管理 |
| `tuple` | 元组 |
| `cerrno` | 里面包含了`error.h`,定义了`errno`的整数值 |
| `bits/functexcept.h` | 定义了一些内部函数,与抛出异常有关 |
| `bits/functional_hash.h` | 定义了一些哈希相关的类,`hash`函数最终调用了`bits/hash_bytes.h`的`_Hash_bytes` |
| `bits/invoke.h` | 实现了`__invoke`类,用来调用某个可调用对象 |
| `bits/gthr.h` | 引入了`bits/gthr-default.h`,这个宏里面定义了`thread`所需的`posix`类型 |

```cpp
#if defined(_GLIBCXX_HAS_GTHREADS)
```
在`bits/c++config.h`中定义如果`gthread`库可用,值为`1`.

### 可见性

```cpp
namespace std _GLIBCXX_VISIBILITY(default)
{
_GLIBCXX_BEGIN_NAMESPACE_VERSION
```

命名空间的可见性为默认,实体在共享库中可见,并且可以被抢占.


这个`_GLIBCXX_VISIBILITY`定义如下:
```cpp
#ifndef _GLIBCXX_PSEUDO_VISIBILITY
# define _GLIBCXX_PSEUDO_VISIBILITY(V)
#endif

#if _GLIBCXX_HAVE_ATTRIBUTE_VISIBILITY
# define _GLIBCXX_VISIBILITY(V) __attribute__ ((__visibility__ (#V)))
#else
// If this is not supplied by the OS-specific or CPU-specific
// headers included below, it will be defined to an empty default.
# define _GLIBCXX_VISIBILITY(V) _GLIBCXX_PSEUDO_VISIBILITY(V)
#endif
```

如果`_GLIBCXX_HAVE_ATTRIBUTE_VISIBILITY`,那么设置可见性,不然他就是空的.

如果宏`_GLIBCXX_INLINE_VERSION`为真,这个`_GLIBCXX_BEGIN_NAMESPACE_VERSION`将会套上一个版本号为名的命名空间,否则啥也不干.


## 内部类

### \_State

```cpp
  /// thread
  class thread
  {
  public:
    // Abstract base class for types that wrap arbitrary functors to be
    // invoked in the new thread of execution.
    struct _State
    {
      virtual ~_State();
      virtual void _M_run() = 0;
    };
    using _State_ptr = unique_ptr<_State>;
```

一个抽象基类`_State`用于封装可执行对象,然后定义一个类型`_State_ptr`,这是一个用来指向`_State`的智能指针.

### 线程id

```cpp
    typedef __gthread_t			native_handle_type;
```

`__gthread_t`在`gthr.h`中被定义为`pthread_t`,这是`linux`的线程id的数据类型,同一进程内唯一.

这个`pthread_t`和`pid_t`有区别.

`pid_t`是标志进程的另一套机制.

众所周知,`linux`的线程是一个特殊的进程,`pid_t`是这个进程的实际的id.

为了将这些线程伪装成同一个进程,当我们使用`getpid()`时,这些线程会返回主线程的进程id.

使用`gettid()`可以获取他们真正的进程id.

这个`pid_t`是全局唯一的.

```cpp
    class id
    {
      native_handle_type	_M_thread;

    public:
      id() noexcept : _M_thread() { }

      explicit
      id(native_handle_type __id) : _M_thread(__id) { }

    private:
      friend class thread;
      friend class hash<thread::id>;

      friend bool
      operator==(thread::id __x, thread::id __y) noexcept;

      friend bool
      operator<(thread::id __x, thread::id __y) noexcept;

      template<class _CharT, class _Traits>
	friend basic_ostream<_CharT, _Traits>&
	operator<<(basic_ostream<_CharT, _Traits>& __out, thread::id __id);
    };

  private:
    id				_M_id;
```

这个内部类有一个`pthread_t`成员,他定义了一些友元函数,以便外部函数使用和比较他的`pthread_id`.

并且重载了`operator<<`以便打印线程id.

```cpp
    // _GLIBCXX_RESOLVE_LIB_DEFECTS
    // 2097.  packaged_task constructors should be constrained
    // 3039. Unnecessary decay in thread and packaged_task
    template<typename _Tp>
      using __not_same = __not_<is_same<__remove_cvref_t<_Tp>, thread>>;
```

`__remove_cvref_t`可以移除`_Tp`类型的`const`,`volatitle`和引用属性.

然后对比去除这些属性的`_Tp`类型和`thread`类型,判断是否相同.

如果相同,那么`__not_same`类型的`value`成员会是`false`,否则为`true`.

## 生命周期相关

### 构造函数

```cpp
  public:
    thread() noexcept = default;

    template<typename _Callable, typename... _Args,
	     typename = _Require<__not_same<_Callable>>>
      explicit
      thread(_Callable&& __f, _Args&&... __args)
      {
	static_assert( __is_invocable<typename decay<_Callable>::type,
				      typename decay<_Args>::type...>::value,
	  "std::thread arguments must be invocable after conversion to rvalues"
	  );
#ifdef GTHR_ACTIVE_PROXY
	// Create a reference to pthread_create, not just the gthr weak symbol.
	auto __depend = reinterpret_cast<void(*)()>(&pthread_create);
#else
	auto __depend = nullptr;
#endif
        _M_start_thread(_S_make_state(
	      __make_invoker(std::forward<_Callable>(__f),
			     std::forward<_Args>(__args)...)),
	    __depend);
      }
```

`_Require`的实现如下
```cpp
  template<bool, typename _Tp = void>
    struct enable_if
    { };

  // Partial specialization for true.
  template<typename _Tp>
    struct enable_if<true, _Tp>
    { typedef _Tp type; };

  template<typename... _Cond>
    using _Require = typename enable_if<__and_<_Cond...>::value>::type;
```

`_Require`的作用:

如果可变模板参数`_Cond`进行`and`运算之后的`::value`为真(也就是这里的`_Callable`的类型和`thread`不同),那么`_Require`类型为合法类型(是一个`void`).

否则他会是一个错误的类型,编译将会报错.

```cpp
	static_assert( __is_invocable<typename decay<_Callable>::type,
				      typename decay<_Args>::type...>::value,
	  "std::thread arguments must be invocable after conversion to rvalues"
	  );
```

判断`_Callable`是否可以执行,根据判断结果,`__is_invocable`最终会继承`true_type`或`false_type`.

对这个`__is_invocable`取`value`,可以得知是否可以调用.

如果不能调用就会报错`std::thread arguments must be invocable after conversion to rvalues`.

#### \_\_make\_invoker


```cpp
#ifdef GTHR_ACTIVE_PROXY
	// Create a reference to pthread_create, not just the gthr weak symbol.
	auto __depend = reinterpret_cast<void(*)()>(&pthread_create);
#else
	auto __depend = nullptr;
#endif
        _M_start_thread(_S_make_state(
	      __make_invoker(std::forward<_Callable>(__f),
			     std::forward<_Args>(__args)...)),
	    __depend);
      }
```

`__make_invoker`会返回一个`_Invoker`对象,函数的实现如下:

```cpp
    template<typename... _Tp>
      using __decayed_tuple = tuple<typename decay<_Tp>::type...>;

    template<typename _Callable, typename... _Args>
      static _Invoker<__decayed_tuple<_Callable, _Args...>>
      __make_invoker(_Callable&& __callable, _Args&&... __args)
      {
	return { __decayed_tuple<_Callable, _Args...>{
	    std::forward<_Callable>(__callable), std::forward<_Args>(__args)...
	} };
      }
  };
```

`__decayed_tuple<typename... _Tp>{__args...}`可以将大括号内的元素包装成一个元组,元祖内每个元素的类型由尖括号内的模板参数决定.

`return`会把这个元祖转换为一个`_Invoker`,这是一个仿函数.后面会提到.

这里是**第一次**移动,这个`_Invoker`位于栈上,构造`_Invoker`的时候参数被移动到`_Invoker`的成员`_M_t`.

#### \_S\_make\_state

```cpp
    template<typename _Callable>
      static _State_ptr
      _S_make_state(_Callable&& __f)
      {
	using _Impl = _State_impl<_Callable>;
	return _State_ptr{new _Impl{std::forward<_Callable>(__f)}};
      }
```

这个函数接收一个可执行对象`__f`把它包装成一个`_State_impl`对象,并返回指向这个对象的智能指针.

这个`_State_impl`后面会提到.

这个`_State_ptr`是前面定义的指向内部类`_State`的智能指针.

这里是**第二次**移动,这里把可执行对象`_Invoker`移动到堆内存,并且用`_State_ptr`指向他.

这个`_State_ptr`指向的内存将会在线程结束的时候析构,析构时才会释放堆上的这块空间.

#### \_M\_start\_thread

```cpp
  void
  thread::_M_start_thread(_State_ptr state, void (*)())
  {
    const int err = __gthread_create(&_M_id._M_thread,
                                     &execute_native_thread_routine,
                                     state.get());
    if (err)
      __throw_system_error(err);
    state.release();
  }
```

同`state`指针指向的内容创建一个线程,然后释放`state`.

创建之后这个线程的线程id会保存在内部对象`_M_id`的`_M_thread`成员中.

这里`state`只是`release`而没有释放内存.

#### execute\_native\_thread\_routine

`execute_native_thread_routine`是执行的关键,他被系统调用,并传入上文中`state.get()`获得的指针

```cpp
    static void*
    execute_native_thread_routine(void* __p)
    {
      thread::_State_ptr __t{ static_cast<thread::_State*>(__p) };
      __t->_M_run();
      return nullptr;
    }
```

一顿转换之后获得了指向可执行对象的智能指针`__t`.

`__t->_M_tun()`将会执行`_Invoker`的`operator()()`.

**第三次**移动发生在这里.

把堆上的内容移动到栈上,传给`std::__invoke`,从而执行.

### 析构函数

```cpp
    ~thread()
    {
      if (joinable())
	std::terminate();
    }
```

`joinable()`会新建一个`id`,和当前的`_M_id`对比,如果相同,那么说明这个线程没有传入一个可执行对象,或者已经被`join`了.

否则执行`std::terminate`结束进程.详情见[这里](https://zh.cppreference.com/w/cpp/error/terminate).

### 拷贝构造函数和移动构造函数

```cpp
    thread(const thread&) = delete;

    thread(thread&& __t) noexcept
    { swap(__t); }
```

### 拷贝赋值函数和移动赋值函数

```cpp
    thread& operator=(const thread&) = delete;

    thread& operator=(thread&& __t) noexcept
    {
      if (joinable())
	std::terminate();
      swap(__t);
      return *this;
    }

    void
    swap(thread& __t) noexcept
    { std::swap(_M_id, __t._M_id); }
```

移动赋值之前需要先判断是否`terminate`,移动`__t`的内容到当前对象.

## 基本操作

### joinable

```cpp
    bool
    joinable() const noexcept
    { return !(_M_id == id()); }
```

新建一个`id`,和当前的`_M_id`对比,如果相同,那么说明这个线程没有传入一个可执行对象,或者已经被`join`了.

### join

```cpp
    void
    join();
```

实现在`thread.cc`文件中

```cpp
  void
  thread::join()
  {
    int __e = EINVAL;
    if (_M_id != id())
      __e = __gthread_join(_M_id._M_thread, 0);
    if (__e)
      __throw_system_error(__e);
    _M_id = id();
  }
```

如果当前是`joinable`的,就调用`__gthread_join`.

然后`_M_id`恢复为`id()`.

`__gthread_join`定义如下:

```cpp
# define __gthrw_(name) __gthrw_ ## name

static inline int
__gthread_join (__gthread_t __threadid, void **__value_ptr)
{
  return __gthrw_(pthread_join) (__threadid, __value_ptr);
}
```

调用`__gthrw_pthread_join`,传入`_M_id._M_thread`和一个`0`.

我目前没有找到他的实现,猜测应该是`pthread_join`封装,以后找到了再补充.


### detach

```cpp
    void
    detach();
```

他的实现如下

```cpp
  void
  thread::detach()
  {
    int __e = EINVAL;
    if (_M_id != id())
      __e = __gthread_detach(_M_id._M_thread);
    if (__e)
      __throw_system_error(__e);
    _M_id = id();
  }
```

也是和`join`只能找到这里

```cpp
__gthread_detach (__gthread_t __threadid)
{
  return __gthrw_(pthread_detach) (__threadid);
}
```

### 获取id

```cpp
    thread::id
    get_id() const noexcept
    { return _M_id; }

    /** @pre thread is joinable
     */
    native_handle_type
    native_handle()
    { return _M_id._M_thread; }
```

### 返回实现支持的并发线程数量

```cpp
    // Returns a value that hints at the number of hardware thread contexts.
    static unsigned int
    hardware_concurrency() noexcept;
```

实现如下
```cpp
  unsigned int
  thread::hardware_concurrency() noexcept
  {
    int __n = _GLIBCXX_NPROCS;
    if (__n < 0)
      __n = 0;
    return __n;
  }
```

## 内部类 II

### _State_impl


```cpp
  private:
    template<typename _Callable>
      struct _State_impl : public _State
      {
	_Callable		_M_func;

	_State_impl(_Callable&& __f) : _M_func(std::forward<_Callable>(__f))
	{ }

	void
	_M_run() { _M_func(); }
      };
```

这个类继承了`_State`并且有一个可执行对象作为成员.

将`_M_run()`实现为执行这个可执行对象.

在构造的位置,他把接收到的接收到的`_Invoker`,当作`_M_func`.

## 一些内部函数

```cpp
    void
    _M_start_thread(_State_ptr, void (*)());

    template<typename _Callable>
      static _State_ptr
      _S_make_state(_Callable&& __f)
      {
	using _Impl = _State_impl<_Callable>;
	return _State_ptr{new _Impl{std::forward<_Callable>(__f)}};
      }
```

前面构造函数已经讲过.

## 内部类 III

### \_Impl\_base

```cpp
#if _GLIBCXX_THREAD_ABI_COMPAT
  public:
    struct _Impl_base;
    typedef shared_ptr<_Impl_base>	__shared_base_type;
    struct _Impl_base
    {
      __shared_base_type	_M_this_ptr;
      virtual ~_Impl_base() = default;
      virtual void _M_run() = 0;
    };
```

这个`_Impl_base`持有一个指向`_Impl_base`的智能指针.目前意义不明,后面将会提到.

####

```cpp
  private:
    void
    _M_start_thread(__shared_base_type, void (*)());

    void
    _M_start_thread(__shared_base_type);
#endif
```

实现如下

```cpp
#if _GLIBCXX_THREAD_ABI_COMPAT
  void
  thread::_M_start_thread(__shared_base_type __b)
  {
    if (!__gthread_active_p())
#if __cpp_exceptions
      throw system_error(make_error_code(errc::operation_not_permitted),
                         "Enable multithreading to use std::thread");
#else
      __throw_system_error(int(errc::operation_not_permitted));
#endif
    _M_start_thread(std::move(__b), nullptr);
  }
  void
  thread::_M_start_thread(__shared_base_type __b, void (*)())
  {
    auto ptr = __b.get();
    // Create a reference cycle that will be broken in the new thread.
    ptr->_M_this_ptr = std::move(__b);
    int __e = __gthread_create(&_M_id._M_thread,
                               &execute_native_thread_routine_compat, ptr);
    if (__e)
    {
      ptr->_M_this_ptr.reset();  // break reference cycle, destroying *ptr.
      __throw_system_error(__e);
    }
  }
#endif
```

他和前面提到的`thread::_M_start_thread(_State_ptr state, void (*)())`不同.

目前没看出来哪里会用到这个内部类,日后补充.

### \_Invoker

```cpp
  private:
    // A call wrapper that does INVOKE(forwarded tuple elements...)
    template<typename _Tuple>
      struct _Invoker
      {
	_Tuple _M_t;

	template<typename>
	  struct __result;
	template<typename _Fn, typename... _Args>
	  struct __result<tuple<_Fn, _Args...>>
	  : __invoke_result<_Fn, _Args...>
	  { };

	template<size_t... _Ind>
	  typename __result<_Tuple>::type
	  _M_invoke(_Index_tuple<_Ind...>)
	  { return std::__invoke(std::get<_Ind>(std::move(_M_t))...); }

	typename __result<_Tuple>::type
	operator()()
	{
	  using _Indices
	    = typename _Build_index_tuple<tuple_size<_Tuple>::value>::__type;
	  return _M_invoke(_Indices());
	}
      };
```

#### \_M\_invoke

这个函数把`_M_t`中的每个元素取出,传给`std::__invoke`以执行.

然后返回`__result<_Tuple>::type`类型的返回值.

这里的具体实现日后补充.

#### \_\_make\_invoker

```cpp
    template<typename... _Tp>
      using __decayed_tuple = tuple<typename decay<_Tp>::type...>;

  public:
    // Returns a call wrapper that stores
    // tuple{DECAY_COPY(__callable), DECAY_COPY(__args)...}.
    template<typename _Callable, typename... _Args>
      static _Invoker<__decayed_tuple<_Callable, _Args...>>
      __make_invoker(_Callable&& __callable, _Args&&... __args)
      {
	return { __decayed_tuple<_Callable, _Args...>{
	    std::forward<_Callable>(__callable), std::forward<_Args>(__args)...
	} };
      }
  };
```

上面构造函数处已经介绍.

## 线程比较操作

```cpp
  inline void
  swap(thread& __x, thread& __y) noexcept
  { __x.swap(__y); }

  inline bool
  operator==(thread::id __x, thread::id __y) noexcept
  {
    // pthread_equal is undefined if either thread ID is not valid, so we
    // can't safely use __gthread_equal on default-constructed values (nor
    // the non-zero value returned by this_thread::get_id() for
    // single-threaded programs using GNU libc). Assume EqualityComparable.
    return __x._M_thread == __y._M_thread;
  }

  inline bool
  operator!=(thread::id __x, thread::id __y) noexcept
  { return !(__x == __y); }

  inline bool
  operator<(thread::id __x, thread::id __y) noexcept
  {
    // Pthreads doesn't define any way to do this, so we just have to
    // assume native_handle_type is LessThanComparable.
    return __x._M_thread < __y._M_thread;
  }

  inline bool
  operator<=(thread::id __x, thread::id __y) noexcept
  { return !(__y < __x); }

  inline bool
  operator>(thread::id __x, thread::id __y) noexcept
  { return __y < __x; }

  inline bool
  operator>=(thread::id __x, thread::id __y) noexcept
  { return !(__x < __y); }
```

纯粹是在数值上对比线程id.

## 哈希

```cpp
  // DR 889.
  /// std::hash specialization for thread::id.
  template<>
    struct hash<thread::id>
    : public __hash_base<size_t, thread::id>
    {
      size_t
      operator()(const thread::id& __id) const noexcept
      { return std::_Hash_impl::hash(__id._M_thread); }
    };
```

关键的函数`_Hash_impl::hash`定义于`bits/functional_hash.h`

```cpp
  struct _Hash_impl
  {
    static size_t
    hash(const void* __ptr, size_t __clength,
	 size_t __seed = static_cast<size_t>(0xc70f6907UL))
    { return _Hash_bytes(__ptr, __clength, __seed); }

    template<typename _Tp>
      static size_t
      hash(const _Tp& __val)
      { return hash(&__val, sizeof(__val)); }

    template<typename _Tp>
      static size_t
      __hash_combine(const _Tp& __val, size_t __hash)
      { return hash(&__val, sizeof(__val), __hash); }
  };
```

他最终调用了`_Hash_bytes`.定义于`hash_bytes.cc`

```cpp
  _Hash_bytes(const void* ptr, size_t len, size_t seed)
  {
    static const size_t mul = (((size_t) 0xc6a4a793UL) << 32UL)
                              + (size_t) 0x5bd1e995UL;
    const char* const buf = static_cast<const char*>(ptr);
    // Remove the bytes not divisible by the sizeof(size_t).  This
    // allows the main loop to process the data as 64-bit integers.
    const int len_aligned = len & ~0x7;
    const char* const end = buf + len_aligned;
    size_t hash = seed ^ (len * mul);
    for (const char* p = buf; p != end; p += 8)
      {
        const size_t data = shift_mix(unaligned_load(p) * mul) * mul;
        hash ^= data;
        hash *= mul;
      }
    if ((len & 0x7) != 0)
      {
        const size_t data = load_bytes(end, len & 0x7);
        hash ^= data;
        hash *= mul;
      }
    hash = shift_mix(hash) * mul;
    hash = shift_mix(hash);
    return hash;
  }
```

## 打印线程

```cpp
  template<class _CharT, class _Traits>
    inline basic_ostream<_CharT, _Traits>&
    operator<<(basic_ostream<_CharT, _Traits>& __out, thread::id __id)
    {
      if (__id == thread::id())
	return __out << "thread::id of a non-executing thread";
      else
	return __out << __id._M_thread;
    }
```

## this thread

以下是一些获取id,睡眠相关的函数.

```cpp
  namespace this_thread
  {
    /// get_id
    inline thread::id
    get_id() noexcept
    {
#ifdef __GLIBC__
      // For the GNU C library pthread_self() is usable without linking to
      // libpthread.so but returns 0, so we cannot use it in single-threaded
      // programs, because this_thread::get_id() != thread::id{} must be true.
      // We know that pthread_t is an integral type in the GNU C library.
      if (!__gthread_active_p())
	return thread::id(1);
#endif
      return thread::id(__gthread_self());
    }

    /// yield
    inline void
    yield() noexcept
    {
#ifdef _GLIBCXX_USE_SCHED_YIELD
      __gthread_yield();
#endif
    }

    void
    __sleep_for(chrono::seconds, chrono::nanoseconds);

    /// sleep_for
    template<typename _Rep, typename _Period>
      inline void
      sleep_for(const chrono::duration<_Rep, _Period>& __rtime)
      {
	if (__rtime <= __rtime.zero())
	  return;
	auto __s = chrono::duration_cast<chrono::seconds>(__rtime);
	auto __ns = chrono::duration_cast<chrono::nanoseconds>(__rtime - __s);
#ifdef _GLIBCXX_USE_NANOSLEEP
	__gthread_time_t __ts =
	  {
	    static_cast<std::time_t>(__s.count()),
	    static_cast<long>(__ns.count())
	  };
	while (::nanosleep(&__ts, &__ts) == -1 && errno == EINTR)
	  { }
#else
	__sleep_for(__s, __ns);
#endif
      }

    /// sleep_until
    template<typename _Clock, typename _Duration>
      inline void
      sleep_until(const chrono::time_point<_Clock, _Duration>& __atime)
      {
	auto __now = _Clock::now();
	if (_Clock::is_steady)
	  {
	    if (__now < __atime)
	      sleep_for(__atime - __now);
	    return;
	  }
	while (__now < __atime)
	  {
	    sleep_for(__atime - __now);
	    __now = _Clock::now();
	  }
      }
  }
```

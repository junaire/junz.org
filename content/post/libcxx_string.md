---
title : "浅析 libc++ 中的 string 实现"
author : "Jun"
date: 2022-08-13T13:33:39+08:00
---


`std::string` 也许是 C++ 程序员最常用的标准库数据结构之一了，用了这么久的 `std::string`，它内部究竟是如何实现的？究竟什么是`SSO (Small String Optimization)`？最近刚好对 `libc++` 的代码感兴趣，就以它的实现为例，简要的回答下以上问题。**（注意为了方便叙述，我对展示的代码进行了一点裁减，并非为实际代码！）**



## 内部结构

我这里的 libc++ 版本为 [37db2833](https://github.com/llvm/llvm-project/commit/37db283362232eaa0a57d452fee45cf2b147f356) 。可以看到，`libcxx` 内部定义了一个宏（`_LIBCPP_ABI_ALTERNATE_STRING_LAYOUT`），这个宏由 [a66a7b30](https://github.com/llvm/llvm-project/commit/a66a7b30ce3624a849867ad39ac62fb8944eb141) 引入，简单来说就是可以通过它控制一些会造成 `ABI-changing` 的特性，比如我们这里对于 `std::string` 的内部布局。下面这段代码简单展示了这个宏的作用：

```c++
#ifdef _LIBCPP_ABI_ALTERNATE_STRING_LAYOUT
    struct __long
    {
        pointer   __data_;
        size_type __size_;
        size_type __cap_ : sizeof(size_type) * CHAR_BIT - 1;
        size_type __is_long_ : 1;
    };
    enum {__min_cap = (sizeof(__long) - 1)/sizeof(value_type) > 2 ?
                      (sizeof(__long) - 1)/sizeof(value_type) : 2};
    struct __short
    {
        value_type __data_[__min_cap];
        unsigned char __padding_[sizeof(value_type) - 1];
        unsigned char __size_ : 7;
        unsigned char __is_long_ : 1;
    };
#else
    // Attribute 'packed' is used to keep the layout compatible with the
    // previous definition that did not use bit fields. This is because on
    // some platforms bit fields have a default size rather than the actual
    // size used, e.g., it is 4 bytes on AIX. See D128285 for details.
    struct __long
    {
        struct _LIBCPP_PACKED {
            size_type __is_long_ : 1;
            size_type __cap_ : sizeof(size_type) * CHAR_BIT - 1;
        };
        size_type __size_;
        pointer   __data_;
    };
    enum {__min_cap = (sizeof(__long) - 1)/sizeof(value_type) > 2 ?
                      (sizeof(__long) - 1)/sizeof(value_type) : 2};
    struct __short
    {
        struct _LIBCPP_PACKED {
            unsigned char __is_long_ : 1;
            unsigned char __size_ : 7;
        };
        char __padding_[sizeof(value_type) - 1];
        value_type __data_[__min_cap];
    };
#endif // _LIBCPP_ABI_ALTERNATE_STRING_LAYOUT
```

可以看到不管选取的哪个 `layout`，我们都定义了两个结构体。其中结构体 `__long` 负责储存长字符串，他维护了一个指向堆的指针 `__data_`，负责存储实际的字符串，而结构体 `__short` 则负责存储短字符串，也就是所谓的 `SSO`。由于堆分配较为昂贵，我们将较短的字符串直接用一个字符数组储存在栈上，也就是这里的 `__data__[__min_cap]`。GDB 调试下可以发现 `__min_cap` 的大小为 23。也就是小于 23 的字符会被内联在 `std::string` 中，大于这个数的字符串会被储存在堆中。



接着，以下代码定义了总的内部储存结构：

```c++
    union __ulx{__long __lx; __short __lxx;};

    enum {__n_words = sizeof(__ulx) / sizeof(size_type)};

    struct __raw
    {
        size_type __words[__n_words];
    };

    struct __rep
    {
        union
        {
            __long  __l;
            __short __s;
            __raw   __r;
        };
    };

    __compressed_pair<__rep, allocator_type> __r_;
```

可以看到 `__r_` 为实际维护字符串的成员，它的类型是一个 `_compressed_pair`，我们可以简单将他理解为是一个 `std::pair` 。`__r_` 的第一个成员是 `__rep`，第二个为`allocator`，也就是负责实际对内存进行操作和分配的结构。而 `__rep` 内只有一个 `union` ，其中成员可能为前面所说的长字符串 `__long`，短字符串 `__short`，或者为 `__raw`。这个 `__raw` 好像没啥用，也不知道是干啥的。



## 字符串的构造

下面通过一段简单的代码看一下 `string` 是如何构造的：

```c++
std::string str{};
```

它选择了下面这个构造函数：

```c++
template <class _CharT, class _Traits, class _Allocator>
inline basic_string<_CharT, _Traits, _Allocator>::basic_string()
     : __r_(__default_init_tag(), __default_init_tag())
{
    __default_init();
}
```

其中，`__default_init_tag` 为一个空的结构体，定义在`compressed_pair.h` 中：

```c++
 struct __default_init_tag {}
```

而 `__default__init` 的具体实现如下：

```c++
void __default_init() {
  __zero();
  if (__libcpp_is_constant_evaluated()) {
   // ...
   }
}
```

其中 `__libcpp_is_constant_evaluated()` 中的代码与 `constexpr string` 有关，我们不必深究。而 `__zero()` 便为我们真正初始化的代码。

```c++
void __zero() {
   __r_.first() = __rep();
}
```

 其实也就是将 `__r_` 的第一个成员 `__rep` 默认初始化，没啥好说的。



再看一段更常见更有意义的代码：

```c++
std::string str("Hello, world!");
```



通过挂上 `gdb` 进行动态调试我们可以发现它调用了以下构造函数：

```c++
template <class = __enable_if_t<__is_allocator<_Allocator>::value, nullptr_t> >
basic_string(const _CharT* __s) : __r_(__default_init_tag(), __default_init_tag()) {
   __init(__s, traits_type::length(__s));
}
```



这段代码真正需要注意的也就是 `__init(__s, traits_type::length(__s));` 这句函数调用。而它的实现为：

```c++
template <class _CharT, class _Traits, class _Allocator>
void
basic_string<_CharT, _Traits, _Allocator>::__init(const value_type* __s, size_type __sz)
{
    pointer __p;
    if (__fits_in_sso(__sz))
    {
        __set_short_size(__sz);
        __p = __get_short_pointer();
    }
    else
    {
        auto __allocation = std::__allocate_at_least(__alloc(), __recommend(__sz) + 1);
        __p = __allocation.ptr;
        __begin_lifetime(__p, __allocation.count);
        __set_long_pointer(__p);
        __set_long_cap(__allocation.count);
        __set_long_size(__sz);
    }
    traits_type::copy(std::__to_address(__p), __s, __sz);
    traits_type::assign(__p[__sz], value_type());
}
```



这里比较重要我们逐行进行讲解。首先我们判断长度是否能塞进短字符串（`__fits_in_sso：__sz < __min_cap`），如果可以的话我们就调用 `__set_short_size`，即设置结构体 `__short` 的相关成员：

```c++
void __set_short_size(size_type __s) {
  __r_.first().__s.__size_ = __s;
  __r_.first().__s.__is_long_ = false;
}
```

最后拿到前面提到的短字符串数组 `__data_`的指针。

如果字符串很长没办法内联在 `string` 中，我们就要就行一个动态分配，将字符串储存在堆上。我们尤其要注意的就是 `auto __allocation = std::__allocate_at_least(__alloc(), __recommend(__sz) + 1);` 这句调用。它分配了 `__recommend(__sz) + 1` 大小的空间，负责存储字符串（后面的加一是因为还要在末尾在放一个`NULL terminator`）。而 `__recommend` 调用又是啥？难道我们不是只分配 `__sz + 1` 大小的空间就足够了吗？ 事实证明这远比我们想得要复杂：



```c++
template <size_type __a> static
size_type __align_it(size_type __s)
{return (__s + (__a-1)) & ~(__a-1);}

enum {__alignment = 16};

static size_type __recommend(size_type __s) {
   if (__s < __min_cap) {
     return static_cast<size_type>(__min_cap) - 1;
    }
    size_type __guess = __align_it<sizeof(value_type) < __alignment ?
                        __alignment/sizeof(value_type) : 1 > (__s+1) - 1;
    if (__guess == __min_cap) ++__guess;
      return __guess;
}
```

这段代码太复杂我懒得写了，反正就是考虑到了 alignment 的问题。



最后，将我们的字符串拷贝进 `__p` 中，也就是指向对应实际储存字符串的指针。当然别忘了还要将最后一个字符设置为 `NULL` 终止符。



## 动态扩容

在上面的讨论中，我们只考虑了字符串为短字符串或者长字符串的情形，考虑以下代码：

```c++
std::string str("1234567890");
str.append("1234567890abcd");
```

当我们尝试将另一段字符串 `append` 进 `std::string` 时，如果总的长度超过了 `__min_cap` （也就是23）该怎么办呢？

再次借助动态调试，我们可以发现我们进入了这个函数：

```c++
template <class _CharT, class _Traits, class _Allocator>
basic_string<_CharT, _Traits, _Allocator>&
basic_string<_CharT, _Traits, _Allocator>::append(const value_type* __s, size_type __n)
{
    size_type __cap = capacity();
    size_type __sz = size();
    if (__cap - __sz >= __n)
    {
        if (__n)
        {
            value_type* __p = std::__to_address(__get_pointer());
            traits_type::copy(__p + __sz, __s, __n);
            __sz += __n;
            __set_size(__sz);
            traits_type::assign(__p[__sz], value_type());
        }
    }
    else
        __grow_by_and_replace(__cap, __sz + __n - __cap, __sz, __sz, 0, __n, __s);
    return *this;
}
```



可以看到，当我们剩余的容量不足以储存想添加的字符串时，我们调用了 `__grow_by_and_replace` 这个函数：

```c++
template <class _CharT, class _Traits, class _Allocator>
void
basic_string<_CharT, _Traits, _Allocator>::__grow_by_and_replace
    (size_type __old_cap, size_type __delta_cap, size_type __old_sz,
     size_type __n_copy,  size_type __n_del,     size_type __n_add, const value_type* __p_new_stuff)
{
    size_type __ms = max_size();
    pointer __old_p = __get_pointer();
    size_type __cap = __old_cap < __ms / 2 - __alignment ?
                          __recommend(std::max(__old_cap + __delta_cap, 2 * __old_cap)) :
                          __ms - 1;
    auto __allocation = std::__allocate_at_least(__alloc(), __cap + 1);
    pointer __p = __allocation.ptr;
    __begin_lifetime(__p, __allocation.count);
    if (__n_copy != 0)
        traits_type::copy(std::__to_address(__p),
                          std::__to_address(__old_p), __n_copy);
    if (__n_add != 0)
        traits_type::copy(std::__to_address(__p) + __n_copy, __p_new_stuff, __n_add);
    size_type __sec_cp_sz = __old_sz - __n_del - __n_copy;
    if (__sec_cp_sz != 0)
        traits_type::copy(std::__to_address(__p) + __n_copy + __n_add,
                          std::__to_address(__old_p) + __n_copy + __n_del, __sec_cp_sz);
    if (__old_cap+1 != __min_cap || __libcpp_is_constant_evaluated())
        __alloc_traits::deallocate(__alloc(), __old_p, __old_cap+1);
    __set_long_pointer(__p);
    __set_long_cap(__allocation.count);
    __old_sz = __n_copy + __n_add + __sec_cp_sz;
    __set_long_size(__old_sz);
    traits_type::assign(__p[__old_sz], value_type());
}
```

这段代码是真的复杂，我稍微将它翻译成人话：

1. 通过计算，拿到新的 `__cap` 并进行分配，`__p` 是指向新的堆内存的指针。
2. 将第一段旧字符串拷贝到新分配的 `__p` 中。
3. 将我们想要 `append` 进的第二段字符串接在 `__p` 的后面。
4. 如果我们旧的字符串的容量大于 `__min_cap`，即为长字符串，释放内存。
5. 在 `__long` 中设置新的相关的成员，如储存字符串的指针，长度和容量。
6. 最后在字符串最后一位加上 `NULL` 终止符。



## 写在后面

可以看到其实 `libcxx` 的代码读起来还是相当舒服和轻松的，后面我可能还会写写其他标准库设施的代码解读。

最后说一下如何在 Linux 下使用 Clang 和自己编译的 `libcxx` 编译 C++ 代码：

```bash
# 编译 libcxx
cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo   \
      -DCMAKE_EXPORT_COMPILE_COMMANDS=YES \
      -DCMAKE_C_COMPILER=clang            \
      -DCMAKE_CXX_COMPILER=clang++        \
      -DLLVM_USE_LINKER=lld               \
      -S runtimes                         \
      -B build_libcxx                     \
      -G Ninja                            \
      -DLIBCXX_ENABLE_ASSERTIONS=ON       \
      -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind" \
ninja -C build_libcxx
      

# 使用 libcxx
clang++ -nostdinc++ -nostdlib++                         \
          -isystem /path/to/build_libcxx/include/c++/v1 \
          -L /path/to/build_libcxx/lib                  \
          -Wl,-rpath, /path/to/build_libcxx/lib         \
          -lc++                                         \
          -std=c++20                                    \
          test.cc -g -O0
```


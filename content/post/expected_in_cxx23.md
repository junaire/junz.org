---
title: "std::expected 基本使用"
author: "Jun"
date: 2022-05-09T23:52:21+08:00
---

最近看cppreference的编译器支持的时候发现libstdc++已经实现`<expected>`了，可惜的是在网上看了一下发现没什么太多它的材料，只能自己看paper和实现写篇博文记录下学习过程了。

## 什么是`std::expected`?

简单点理解它和`std::optional`差不多，但是`std::optional`只能表示有正常的值或者为`std::nullopt`，即空值。而`std::expected`则可以表示一个期望的值和一个错误的值，相当于两个成员的`std::variant`，但是在接口上更方便使用。可以把它当作新的一种的错误处理方式。



## 基本使用

### 构造
`std::expected<T,E>`有两个模板参数，第一个表示为期望的值，第二个表示错误的值。

如果是期望的值，则有一个隐式的转换。

```c++
std::expected<int, std::string> e = 42;
```

如果是异常值，则要通过`std::expected()`来初始化。
```c++
std::expected<int, std::string> e = std::unexpected("Error");
```

#### 默认构造函数
```c++
std::expected<S, int> e;
```
要求`S`有一个`default constructor`，否则会报错。

### 使用

和`std::optional`一样有指针语义，可以解引用。

```c++
std::expected<int, std::string> e = 42;
if (e)
  std::cout << *e << "\n"; // prints 42
```

注意解引用前必须检查，否则为**UB!!!**



下面和上面效果上等价：

```c++
std::expected<int, std::string> e = 42;
if (e.has_value())
  std::cout << e.value() << "\n"; // prints 42
```

注意`std::expected<T,E>::value()`在值不存在会抛出一个异常，一定程度上更安全。

### 表示错误

```c++
std::expected<int, std::string> e = std::unexpected("Error");
if (!e.has_value())
  std::cout << e.error() << "\n"; // prints `Error`
```

如果没有错误使用的话为**UB!!!**



如果说返回的结果为错误，可以通过`value_or()`来更优雅地提供一个默认值。

```c++
std::expected<int, std::string> MayHasErr(int i) {
    if (i < 0)
        return std::unexpected("Error");
    else
        return i * 2;
}

int main() {
    std::cout << MayHasErr(3).value() << "\n"; // print 3
    std::cout << MayHasErr(-2).value_or(42) << "\n"; // print 42
}
```

## 内部实现
`std::expected<T,E>`的实现比较简单，不需要依靠任何新的语言特性。它由一个类型为`T`的期望值和`std::unexpected<E>`的union和一个表示是否为错误的`bool`值组成。剩下的就是一些`helper`函数。



```c++
template<class T, class E>
  class expected {
  using unexpected_type = unexpected<E>;
   private:
    bool has_val;
    union {
      T val; // 表示期望的值
      E unex; // 表示异常的值
    };
  };
```



## 参考

- [libstdc++的实现](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/std/expected)

- [P0323R11](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p0323r11.html)


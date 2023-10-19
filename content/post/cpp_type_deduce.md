---
title: "读Effetive Modern C++ 之类型推导"
author: "Jun"
date: 2021-08-11T21:33:43+08:00
---

# 概览
C++98 => 一套规则

C++11 => 新加两套，一套用于`auto`，一套用于`decltype`

C++14 => 拓展了语境

## 模板推导

```c++
template<typename T>
void f(ParamType param) {}
```

```c++
f(expr)
```

T推导的结果，不仅与实参expr有关，还与ParamType的形式有关系

### ParamType 为指针或引用

推导过程中会忽略其引用性。

### ParamType 为为万能引用

- 如果expr为左值，会被推导为左值引用
- 如果为右值，则与第一种情况相同

### ParamType 为其它值

推导过程中会忽略其引用性，常量性。

**只忽略顶层const, 如 `const char* const` 会被推导为 `const char*`**



## auto 推导

当某变量使用auto来声明时，auto就扮演了模板中的T，而变量的修饰词则扮演了ParamType

```c++
const auto x = 1;
```

相当于：

```c++
template<typename T>
void f(const T param) {}
```

在大多数情况下，auto推导与模板推导没有区别，除了以下情况：

```c++
auto x {1,3,5,7};
```

对于大括号初始化表达式的处理方式，是auto类型推导和模板类型推导的唯一不同之处。当使用auto时，推导的类型为 `std::initializer_list` 的一个实例。

## decltype 推导

在C++11中，decltype的主要用途就在于声明返回值类型依赖与形参类别的函数模板。

在函数返回类型的推导中，auto 会忽略其引用性，我们可以使用 `decltype(auto)` 来修正这一问题。

如果仅有一个名字的话，其行为保持不变，若是比仅有名字更复杂的左值表达式的话，decltype保证得出的型别总为左值引用。**只要不是T,它就得出T&**

在C++的定义中，`(x)` 也是左值，所以 `decltype((x))` 结果也为左值引用。

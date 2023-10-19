---
title: "C++ 移动语义基础"
author: "Jun"
date: 2023-02-21T16:19:35+08:00
---

## 移动语义的作用

直接看一个例子：
```cpp
#include <iostream>

struct S {
  S() { std::cout << "S()\n"; }
  S(const S&) { std::cout << "S(const S&)\n"; }
  ~S() { std::cout << "~S()\n"; }
};

S foo() {
  return S();
}

int main() {
  foo();
}
```

用以下命令编译：
```
clang++ demo.cc -std=c++03 -fno-elide-constructors
```

结果为：
```
S()
S(const S&)
~S()
~S()
```

注意到我们有一次构造函数的调用，一次拷贝构造函数的调用，两次析构函数的调用。这是因为我们以值的形式返回，首先会先构造出一个临时的 `S` 对象，然后将它复制给函数的返回值，最后这个临时对象与函数返回对象依次析构。可以想象到如果 `S` 是一个很大的对象时，这将是一个不小的开销。

我们修改代码为：
```cpp
int main() {
  S s = foo();
}
```

编译运行后的结果为：
```
S()
S(const S&)
~S()
S(const S&)
~S()
~S()
```

可以看到我们又多了一次拷贝构造，也就是说我们把函数返回的值又拷贝给了 `main` 中的 `s`。这显然是很荒唐的，这么一段代码中竟然存在两次非必要拷贝。难道我们就不能直接把 `return S()` 中构造的对象直接初始化给 `main` 中的 `s`？

在 C++11 引入移动语义后，我们可以给 `S` 增加一个移动构造函数：
```cpp
S(S&&) { std::cout << "S(S&&)\n"; }
```

用以下命令编译：
```
clang++ demo.cc -std=c++11 -fno-elide-constructors
```

结果为：
```
S()
S(S&&)
~S()
S(S&&)
~S()
~S()
```

可以看到之前所有的拷贝构造函数都被现在的移动构造函数所取代了。如果我们自定义我们的移动构造函数，让它直接使用原来对象的所有资源，同时将原来对象持有的资源置为空，那就相当于“偷走了”原来对象的资源，使得原本昂贵的复制操作变成了廉价的移动操作。

例如类 `S` 持有一个指针，指向一块堆内存：
```cpp
struct S {
  const char* data;
  int len;
}
```

对于它的移动构造函数，我们可以有如下实现：
```cpp
S::S(S&& other) {
  data = std::exchange(other.data, nullptr);
  len = std::exchange(other.len, 0);
}
```
这相当于将之前的资源拱手让人，原来的对象进入了一个默认的可析构状态。

但是值得注意的是，我们并没有说将 `return S()` 中构造的对象直接初始化给 `main` 中的 `s`，只是原来的拷贝构造函数变成了移动构造函数。而下面会提到的 `copy elision` 会有进一步优化。

## 移动构造被调用的时机

简单来讲，对于“临时量”，需要被拷贝时，就会调用移动构造函数。例如：
```cpp
S s = S();
```
用以下命令编译：
```
clang++ demo.cc -std=c++11 -fno-elide-constructors
```

结果为：
```
S()
S(S&&)
~S()
~S()
```
因为等号右边的 `S()` 是临时创建出来的，运行到下一行就被销毁了。所以我们拷贝它的时候可以移动它，直接取它的所有资源为己用，反正它后面也没用了，对吗？:)

## 右值与 std::move

而这种“临时量”，被称为右值（`rvalue`）。右值是值类别（value categories）的一种，而值类别又是表达式（expression）的属性之一。认识到值类别是 expression 的属性这一点非常重要，比如：
```cpp
int x = 42;
```
这里如果我们要讨论上面代码的值类别就是不成立的，因为这是一个 `declaration`。C++ 标准中对于表达式的定义是：
> An expression is a sequence of operators and operands that specifies a computation. An expression can result in a value and can cause side effects.

下面举几个例子：
```cpp
42 // integer literal
"Hello, world!" // string literal
nullptr // nullptr literal
x = 42 // binary expression
x // declaration reference expression
```
这些都属于 `expression`，注意结尾没有 `;`，否则便成为了 `statement`。

`expression` 要么会返回一个值 `value`，要么会有副作用。比如上面的 `42`，`"hello, world!"` 和 `x` 都返回了一个值，而 `x = 42` 则产生了副作用。而值类别便描述了值（`value`）的某种属性。

要给出一个清楚易懂的关于右值的定义很难，但是我们暂时可以将其理解是“临时的”的值。拷贝这样的右值就会调用移动构造函数，为我们减少不必要的拷贝**创造了空间。**

那能不能移动一个非右值呢？

假设我们有如下代码：
```cpp
S s1{};
s1.do_something();
S s2 = s1; // 不需要 s1 了，能否将它的资源都转手给 s2？
```
答案是可以的，我们可以将其转为一个右值引用：
```cpp
S s2 = static_cast<S&&>(s1);
```

标准库为此专门提供了一个库函数 `std::move()` 将这一类型转换封装起来，于是我们可以写：
```cpp
S s2 = std::move(s1);
```

由此我们可以发现 `std::move()` 并没有做任何“移动”，它只是一个类型转换，使得移动构造函数有机会调用。如果说我们将移动构造函数也实现为拷贝，那么即使“移动”也将不会有任何性能提升！

另外注意 `s1` 被移动后的声明周期并没有结束，而是进入了称为 `valid but not specified` 的状态。它不能再被使用，除非被赋了新的值，使用一个已经移动过的值，是 C++ 编程中一种常见错误，称为 use after move。

## 值类别

上面我们非常简单地介绍了值类别的一种 -- 右值（`rvalue`），下面我们详细讨论下 C++ 中所有的值类别。
![cpp_value_categories](/images/cpp_value_categories.jpg)

可以看到 C++ 的值类别其实非常混乱，`glvalue` 实际上与 `rvalue` 有一部分重合了。细分下来有三种值类别：
* `lvalue` locator value
* `prvalue` pure readable value
* `xvalue` expiring value

具体的分类标准十分复杂，可见 [Value categories](https://en.cppreference.com/w/cpp/language/value_category)。但简单来说：

* 所有有名字的表达式为 `lvalue`
* 所有字符串字面量，如 `"Hello, world"`，为 `lvalue`
* 所有非字符串字面量，如 `0`，`true` 或 `nullptr`，为 `prvalue`
* 所有没有名字的临时值，尤其是以值返回的对象，为 `prvalue`
* 所有 `std::move()` 的对象，为 `xvalue`

而 C++17 之后，完整类型的 `prvalue` 都可以被自动转换为相同类型的 `xvalue`。这意味着我们可以将 `prvalue` 以 `xvalue` 的形式传递，即使他没有拷贝构造函数或移动构造函数。例如：
```cpp
struct S {
  S() = default;
  S(const S&) = delete;
  S(S&&) = delete;
};

S foo() {
  return S(); // C++17 后合法。
}

void bar(S s) {}

int main() {
  bar(foo()); // C++17 后合法。
}
```

## 右值引用与重载决议

为了配合右值（`rvalue`）的概念，C++ 标准又引入了右值引用这一概念：
```cpp
int&& rvalue_ref = 42;
```
右值引用一般情况下并不会单独使用，可以理解为是右值的附属概念。其中需要注意的一点就是右值引用只能绑定到右值上。下面是它对重载决议的影响（数字代表被调用的顺序）：

|           Call            | `f(X&)` | `f(const X&)` | `f(X&&)` | `f(const X&&)` |
| :-----------------------: | :-----: | :-----------: | :------: | :------------: |
|        `f(value)`         |    1    |       2       |    no    |       no       |
|      `f(const_val)`       |   no    |       1       |    no    |       no       |
|         `f(X{})`          |   no    |       3       |    1     |       2        |
|   `f(std::move(value))`   |   no    |       3       |    1     |       2        |
| `f(std::move(const_val))` |   no    |       2       |    no    |       1        |


总结一下就是以右值引用为函数参数的函数只能接受非 const 的右值引用，而常量左值引用可以接受任何值，不过接收右值的优先级最低，可以看作是一种 fallback。上表还包含了常量右值引用，但是实际上它在语法上没有任何意义，我们可以忽略它，任何时候我们都不应该创建一个常量右值引用。因为右值引用生来是为了“移动”，为了转移资源而服务的，又加了“常量”，即不可变，便没有意义了。

## Copy elision

`copy elision` 是编译器的一种优化，用于消除不必要的拷贝或者移动操作。它的两种情况更常见的名字是 `NRVO (Named Return Value Optimization)` 或者 `RVO (Return Value Optimization)`。在 C++17 后，这种优化是义务的，或者说是标准保证一定会发生的。

在一个返回语句中，当操作数是一个与函数返回类型相同的类型（忽略 CV qualification）的 `prvalue`，`copy elision` 一定会发生，这里一般被称为 `RVO`。
```cpp
S foo() {
  return S(); // RVO C++17 后一定会在这里发生。
}
int main() {
  S s = foo(); // copy elision C++17 后一定会在这里发生。
}
```

用以下命令编译：
```
clang++ demo.cc -std=c++17
```
输出：
```
S()
~S()
```
可以看到我们实现了把 `return S()` 中构造的对象直接初始化给 `main` 中的 `s`，再也没有任何临时量的产生了！这也是为什么前文要用 `-fno-elide-constructors` 编译，因为现在的编译器都有这个优化，我们只有关掉它才能看到移动语义的效果。

下面的情况下不保证会发生，但一般编译器都实现了：
```cpp
S foo() {
  S s{};
  return s; // NRVO 不保证但一般会发生在这里。
}
int main() {
  S s = foo(); // copy elision C++17 后一定会在这里发生。
}
```
输出：
```
S()
~S()
```
---
title: "C++模板基础"
author : "Jun"
date: 2022-02-27T20:39:40+08:00
---

## 函数模板

先看几个简单的例子了解下函数模板是如何使用的：

```c++
template<typename T>
T max(T a, T b)
{
  return b < a ? a : b;
}

max(1,2); // 隐式指定了模板实参为int，由编译器推导
max<int>(4,2); // 显式指定了模板实参为int

template<int I>
int add() {
    return I + 42;
}

add<42>();
```

在尖括号间是模板参数列表（template parameter list），其中元素称为模板参数（template parameter），而T 是类型参数，表示任意类型。

当我们调用了一个函数模板时，我们隐式或显式指定了模板实参（template argument），将其绑定到模板参数上。

### 模板类型参数

在上面的例子中的`max`函数中，有一个模板类型参数（type parameter）。我们可以将类型参数看作类型说明符，可以用它来指定返回类型或函数的参数类型。

#### 类型推断中的类型转换
- 如果调用参数是按引用传递的，任何类型都不被转换
- 如果调用参数是按值传递的，那么只有退化（decay）这类转换是被允许的。

注意类型推断并不适用于默认调用参数，如：
```c++
template<typename T>
void f(T = ""){}

f(); // Error

template<typename T = std::string>
void g(T = ""){}
g() // OK
```



#### 非类型参数

模板参数不一定要是某种具体的类型，也可以是常规数值。比如上面例子中的`add`函数。对于非类型参数，C++17后可以不用制定其类型而是用`auto`进行推导。
以下代码在C++17中也是成立的：

```c++
template<auto X>
auto echo(){
    return X;
}
```

## 函数模板的重载

- 如果有一个非模板函数和一个与其同名的函数模板共存，并且这个模板可以被实例化为与非模板函数具有相同类型的调用参数，在其他因素完全相同的情况下，优先选择非模板函数。
- 如果模板可以实例化出一个更匹配的函数，选择此模板。
- 可以显式指定一个空的模板列表，这表明它会被解析成一个模板调用。




## 类模板

```c++
template<typename T>
class Stack {
  std::vector<T> elems;
public:
  void push(const T& elem);
  const T& top() const;
};

Stack<int> IntStack;
```

与函数模板不同，编译器不可以隐式推导出模板参数类型，所以在使用类模板时，我们必须要在模板名后的尖括号内显式提供模板实参。在类模板内部，T可以像普通类型一样被用来声明成员变量和成员函数。在类内部，Stack就表示这个类，但在外部使用时，必须写`Stack<T>`。

### 成员模板

无论是普通类还是模板类都可以包含本身是模板的成员函数。这种成员被称为成员模板（*`member template`*）。

对于类模板，类和成员各自有自己独立的模板参数。而为了实例化一个类模板的成员模板，我们必须同时提供类和函数模板的实参。



### .template

有时，在调用一个成员模板的时候，有必要显式地指出模板参数。此时，必须使用`.template`来告诉编译器`<`后面是一个模板参数列表。具体可以看下面这个例子：

```c++
#include <bitset>
#include <iostream>

template <unsigned long N>
void printBitset(std::bitset<N> const& bs) {
    std::cout << bs.template to_string<char, std::char_traits<char>,
				       std::allocator<char>>();
}

int main()
{
	std::bitset<5> bit;
	printBitset(bit);
}


```



### 模板类的全特化

为了特化一个模板，在类模板声明的前面改为`template<>`，并且需要指明所需要特化的类型。这些用于特化类模板的类型被用作模板参数，并且需要紧跟在类名的后面：

```c++
template<>
class Stack<std::string>{
...
};
```

所有其它出现T的地方，都应该被特化为类模板的类型。


### 偏特化

类模板可以被部分特化，这样就可以为某些其它特殊情况提供特殊的实现。比如：

```c++
template<typename T>
class Stack<T*> {
  std::vector<T*> elem;
public:
  T* top const;
};
```

注意特化后的函数接口可能不同，比如`top()`。


### 类型别名

类型别名有两种形式：

- `typedef Stack<int> IntStack`
- `using IntStack = Stack<int>`

这一过程称为*`alias declaration`*。 在这种情况下我们都只是为一个已存在的类型定义了一个别名，并没有定义新的类型。新的名字称为*`type alias`*。

### 别名模板

*`alias declaration`*也可以被模板化，这样就可以给一组类型取一个方便的名字。

```c++
template<typename T>
using DequeStack = Stack<T, std::deque<T>>
```

### 类成员的别名模板

```c++
template<typename T>
struct S {
  using Ty = T;
};

template<typename T>
using MyT = typename S<T>::Ty;
```

注意在使用`S<T>::Ty`是必须在前面加上`typename`，向编译器表明这是一个类型，否则编译器默认认为只是一个普通成员变量。

## 变参模板

从C++11开始，模板可以接受一组数量可变的参数，这种模板被称为变参模板（variadic template）

```c++
template<typename T, typename... Args>
void print(T arg1, Args... args) {
    std::cout << arg1 << "\n";
    if constexpr (sizeof...(Args) > 0) // 编译期if
        print(args...);
}
```


### 变参下标

```c++
template<auto... Idx, typename C>
void printRange(const C& coll) {
    print(coll[idx]...);
}

int main() {
    std::vector<int> vec {1,2,3,4,5,6};
    printRange<0,2,3>(vec); // => print(vec[0], vec[2], vec[3])
}
```

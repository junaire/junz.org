---
title: "C++中lambda表达式基础"
author: "Jun"
date: 2021-04-26T21:25:15+08:00
---

## 语法 

### 完整语法

```
[ capture clause ] (parameters) -> return-type  
{   
   definition of method   
} 
```

### 简化语法

在没有形参的情况下，可以简化为：

```
[] { codes here... }
```
## 本质

lambda本质是一个函数对象，可以看作是重载了函数调用运算符的对象。

```c++
struct absInt {
  int operator()(int val) const {
    return val < 0 ? -val : val;
  }
}

absInt a;
int abs_a = a(10);
```

### 举例

```c++
int main(){
  int id {0};
  auto f = [id]() { std::cout << id << "\n"; };
  f();
}
```

上面这个lambda在编译器看来将是下面这个情况：

```c++
class __lambda_8_12 {
    public: 
        __lambda_8_12(int& _id) : id{_id} {}
        inline  void operator()() { std::cout << id << "\n"; }
    private: 
        int id;
};
```

## 捕获

- 捕获只能针对于lambda式的作用域内可见的非静态局部变量，包括形参。

- lambda表达式可以直接使用静态变量，而不需要被捕获。


### 默认按值捕获

`[=]`

以下面的代码为例，lambda表达式`f`捕获的`id`是值为0的那一个。

```c++
int main()
{
  int id {0};
  auto f = [=]() {
    std::cout << "id = " << id << "\n";
    std::cout << "id addr inside: "<< &id << "\n";
  };
  
  id = 100;
  std::cout << "id addr outside: " << &id << "\n";
  f();
  return 0;
}
```

下面的代码和上面的区别在于`mutable`。不加`mutable`的情况下，`id`是只读的。

```c++
int main(){
  int id {0};
  auto f = [=]() mutable {
    std::cout << "id = " << id << "\n";
    std::cout << "id addr inside: "<< &id << "\n";
    ++id;//没有mutable的情况下，这段代码会报错。error: read-only variable is not assignable
  };
  
  id = 100;
  std::cout << "id addr outside: " << &id << "\n";
  f();
  f();
  return 0;
}

```
输出
```
id addr outside: 0x7ffe932c0b88
id = 0
id addr inside: 0x7ffe932c0b80
id = 1
id addr inside: 0x7ffe932c0b80
```
可以看到，lambda捕获的id是一个全新的变量，即使我们将id重新赋值为10，它也不受影响。

### 默认按引用捕获 

`[&]`

lambda表达式里的`id`和外部的`id`是同一个变量，改变外部变量的情况下也会影响内部。

```c++
int main(){
  int id {0};
  auto f = [&id]() {
    std::cout << "id = " << id << "\n";
    std::cout << "id addr inside: "<< &id << "\n";
    ++id;
  };
  
  id = 100;
  std::cout << "id addr outside: " << &id << "\n";
  f();
  f();
  return 0;
}
```
输出
```
id addr outside: 0x7ffde0080a68
id = 100
id addr inside: 0x7ffde0080a68
id = 101
id addr inside: 0x7ffde0080a68
```
可以看到，lambda内部捕获的id就是外部的那个，当我们对其重新赋值的时候，lamnda内部也该改变了。

但是这种按引用捕获可能会导致闭包包含局部变量的引用，当局部变量的生命周期结束后，lamnda将失效，从而引起未定义的行为。

### 初始化捕获/广义lambda捕获 (C++14)

在=号的两边，是不同的作用域。在其左边，是lambda的成员变量，而在右边，则是初始化表达式。

```
[a=a]
[a=std::move(a)]
[a=std::unique_ptr<a>()]
```

- 第一个表达式可以看作是在lambda中创建一个id变量，直接用外部的id初始化它。从此二者分离开，互不干扰。
- 第二个表达式可以看作是在lambda中创建一个id变量，用外部的id的右值去初始化它。
- 第三个表达式中，我们甚至可以直接在lambda表达式中初始化id变量。

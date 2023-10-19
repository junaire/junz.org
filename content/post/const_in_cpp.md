---
title: "简要剖析const关键字"
author: "Jun"
date: 2021-03-13T12:46:54+08:00
---

# `const`的作用
`const`关键字保证了我们的变量不会改变，成为常量．

当用在成员函数时，它保证此成员函数不会改变类中的其他成员

# `const`用法

```
const declaration ;
member-function const ;

```

## `const`用在指针上时
这里有两种情况，一个是指针所指的值为常量，另一个是指针本身为常量．

- 指针所指值为常量，这种情况称为底层const

```
int i = 0;
const int *ptr = &i;
```
我们不可以改变`i`的值，但是可以改变`ptr`的指向．

- 指针本身为常量，这种情况称为顶层const

```
int i = 0;
int* const ptr = &i;
```

我们不能改变`ptr`的指向，也就是`ptr`中保存的值，即`i`的地址，但是我们可以通过解引用改变`i`的值.

## `const`用在成员函数时

`const`成员函数无法改变其他成员，但这却存在一个问题，很明显，当成员函数中存在指针变量时，我们无法保证指针所指值不发生改变．因为我们所保证的仅仅只是指针保存的地址，或者说指针本身保持不变而已．

### 死板的`const`成员函数
`const`成员函数无法改变其他成员，所有的．但问题是有些成员的改变对于我们说是允许的，并不影响我们的需求．举`Effective C++`这本书中的一个例子：
```
class CTextBlock
{
public:
    ...
    std::size length() const;
private:
    char *pText;
    std::size_t textLength;
    bool lengthIsValid;
};
std::size_t CTextBlock::length() const
{
    if(!lengthIsValid)
    {
        textLength = std::strlen(pText);
	lengthIsValid = True;
    }
    return textLength;
}
```
上面这个例子实际上会编译错误，因为我们声明了`length`方法为`const`，但却改变了原对象．C++对此的解决办法是引入[`mutable`](https://en.cppreference.com/w/cpp/keyword/mutable)关键字．

我们只要在成员变量声明前加上它，就可以让它处于始终可变的状态，即使是在`const`成员函数中．


## `passed by reference const`
我们知道在C/C++中，函数传参的方式是`passed by value`．也就是说，函数实际是拷贝实参到形参中，函数中的形参与实参实际是两个值．当函数执行完后，它所产生的＂对象＂，如形参等，将被一一销毁．这就意味着我们无法直接改变函数的实参，举个例子：
```
void func(int i)
{
     i = i + 1;
}

int j = 1;

func(j);

printf("%d\n",j); // j = ???
```
实际上`j`的值仍为1，而不是2．因为函数中的`i`根本不是`j`，而只是具有`j`的值的副本！

### C语言的解决办法
C Style的做法是通过指针．传进函数的其实仍然只是实参的拷贝，也就是我们想改变的那个变量的地址，但通过解引用，我们就可以间接访问到那个变量的值，从而改变它．
```
void func_ptr(int* i)
{
    *i = *i + 1;
}
int j = 1;
func_ptr(&j);
printf("%d\n",j); // j = 2
```
### C++的解决办法
我们知道C语言的指针及其强大和灵活，使用不好的话很容易出现内存错误，而且难以调试．由此，C++提供了一种更好的解决办法，即引入了＂引用＂`reference`

引用的本质其实是指针，但用法与指针区别很大，我们可以简单理解为引用是指针的语法糖．

使用引用的优点是很明显的，由于我们可以通过引用直接访问实参中的值，而不需要额外的拷贝工作，大大地提升了性能．
然而直接使用引用却有一个很严重的问题，因为这么做我们可能无意中改变了实参，而我们所需要的仅仅只是读取实参的值而已，而不是改变它．

**更好的方法是使用常量引用，即`passed by reference const`**

通过常量引用，我们不仅省去了拷贝实参带来的性能损失，而且保证了函数仅仅能够读取值而无法作出任何修改．

```
class Foo {
   //large object...
};

void func(const Foo& f)
{
    //read object f...
}
```
通过上述代码我们可以看到，`Foo`是一个＂很大＂的对象，拷贝成本很高，但是直接使用引用又可能改变它的值．此时使用常量引用便完美解决了这两个问题．在任何需要读值而不需要写值的函数传参情况，我们都推荐使用`passed by reference const`．

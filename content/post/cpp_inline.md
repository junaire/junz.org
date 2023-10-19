---
title: "C++ 中 inline 关键字的语义"
author: "Jun"
date: 2023-07-11T23:45:13+08:00
---

C++ 真的是非常地混乱和难学，本文中我会简单谈谈 `inline` 在 C++ 中的含义和作用。

## 优化器眼里的 inline

在这一层语境下，`inline` 是指将在函数调用处将函数体展开，消除掉函数调用。看一个简单的例子:

假如我们有以下代码，且 `foo` 没有被 `inline` (内联)：

```c++
int foo(int x) {return x + 42;}

int bar(int x) { int y = foo(x) + 1; return y;}
```

可以看到编译器（clang trunk）生成的汇编（https://godbolt.org/z/Ebsa8h9Tq）：

```assembly
foo(int):                                # @foo(int)
        push    rbp
        mov     rbp, rsp
        mov     dword ptr [rbp - 4], edi
        mov     eax, dword ptr [rbp - 4]
        add     eax, 42
        pop     rbp
        ret
bar(int):                                # @bar(int)
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     dword ptr [rbp - 4], edi
        mov     edi, dword ptr [rbp - 4]
        call    foo(int)
        add     eax, 1
        mov     dword ptr [rbp - 8], eax
        mov     eax, dword ptr [rbp - 8]
        add     rsp, 16
        pop     rbp
        ret
```

可以看到编译器生成了一条 `call` 指令，表示调用函数 `foo`。

而当我们将代码改成这样时：

```c++
#define INLINE inline __attribute__((always_inline))
INLINE int foo(int x) {return x + 42;}

int bar(int x) { int y = foo(x) + 1; return y;}
```

编译器（clang trunk）生成的汇编为（https://godbolt.org/z/e3MsnvMbE）：
```
bar(int):                                # @bar(int)
        push    rbp
        mov     rbp, rsp
        mov     dword ptr [rbp - 8], edi
        mov     eax, dword ptr [rbp - 8]
        mov     dword ptr [rbp - 4], eax
        mov     eax, dword ptr [rbp - 4]
        add     eax, 42
        add     eax, 1
        mov     dword ptr [rbp - 12], eax
        mov     eax, dword ptr [rbp - 12]
        pop     rbp
        ret
```

可以看到整 `foo` 的整个函数体已经被干掉了，在 `bar` 中直接被替换为了 `add eax, 42` 即源代码中对应的 `x + 42`。

这就是当我们谈优化时 `inline` 的含义，如果某人说某某函数被 `inline` 掉了，他实际的意思是编译器将这个函数在它的所有调用处展开了。

但是！如果认真看一下我们修改后的代码就会发现，并不是加个 `inline` 关键字函数就会被 `inline`（内联）的。这是为什么呢？我们都知道 C 和 C++ 都是非常古老的语言，在早期编译器还不是很成熟时，我们确实可以通过使用 `inline` 关键字告诉编译器请内联这个函数。而随着编译技术的发展，编译器越来越聪明（至少在某些方面），它会自己去判断是否真的需要去内联这个函数。

比如在 Clang 中，优化器采用了一个 Cost model 导向的算法，会根据一些分析得来的信息进行判断，内联一个函数是否是值得的（profitable）。而 `inline` 关键字，则会作为一个 hint，帮助优化器决策。所以，`inline` 关键字一定是对函数内联有影响的！

我们可以直接看 Clang 编译器的源码 [clang/lib/CodeGen/CodeGenModule.cpp](https://github.com/llvm/llvm-project/blob/0d8ec20fbb6342c31d219978f9a2c014141b2347/clang/lib/CodeGen/CodeGenModule.cpp#L2338-L2350) 来验证这点：

```c++
    if (auto *FD = dyn_cast<FunctionDecl>(D)) {
      auto CheckForInline = [](const FunctionDecl *FD) {
        auto CheckRedeclForInline = [](const FunctionDecl *Redecl) {
          return Redecl->isInlineSpecified();
        };
        if (any_of(FD->redecls(), CheckRedeclForInline))
          return true;
        // 和模板有关的代码，略过。
      };
      if (CheckForInline(FD)) {
        B.addAttribute(llvm::Attribute::InlineHint);
      } else if (...)
        ...
      }
    }
```
可以看到，如果我们的 FunctionDecl (函数声明) 是带 `inline` 关键字的，我们会给生成的 IR 加一个 `llvm::Attribute::InlineHint` 的标记。

而在 LLVM 中端的 Inliner 中搜索 `InlineHint`，可以找到 [llvm/lib/Analysis/InlineCost.cpp](https://github.com/llvm/llvm-project/blob/de58b81e5cf3f7514241b86109d6dccd46a74968/llvm/lib/Analysis/InlineCost.cpp#L1890-L1892)：
```c++
  // Adjust the threshold based on inlinehint attribute and profile based
  // hotness information if the caller does not have MinSize attribute.
  if (!Caller->hasMinSize()) {
    if (Callee.hasFnAttribute(Attribute::InlineHint))
      Threshold = MaxIfValid(Threshold, Params.HintThreshold);
```

即使我们不理解 Inliner 整个算法的含义，至少也可以看出加上 `inline` 关键字是有意义的，虽然不是**决定性因素**，但绝对不是一些人说的毫无意义。

## 链接器中的 inline

首先我们先介绍 C++ 的编译模型：

对于 C 和 C++ 来说，它的每个源文件，即 `.c` 和 `.cpp` 文件都是可以被编译器单独编译的，然后链接器会将编译后生成的所有 `.o` 文件链接在一起，形成一个可执行文件。而每个源文件被称为 Translation Unit。注意，**头文件不是 Translation unit**，因为头文件在预处理时发挥作用，此后就没有这个概念了。举个简单的例子：
```c++
// foo.h
int foo(int x);

// bar.cpp
#include "foo.h"
int bar(int x) { return foo(x) + 42; }

// baz.cpp
#include "foo.h"
int baz(int x) { return foo(x) - 42; }
```
这几个文件经过编译器预处理后，我们相当于有下面两个 Translation unit:
```c++
// bar.cpp
int foo(int x); 
// include 会将头文件中所有内容复制粘贴过来

int bar(int x) { return x + 42; }

// baz.cpp
int foo(int x); 
// include 会将头文件中所有内容复制粘贴过来

int baz(int x) { return foo(x) + 1; }
```

而如果我们尝试在头文件中写一个函数定义呢？如果你试过的话就知道编译器会报错，编译不通过。为什么呢？根据上面的模型我们可以知道，`#inlcude` 会把头文件中所有内容复制过来，也就是说我们在两个 Translation unit 都定义了同一个函数。那我们可以想象出编译器会生成类似于下面的代码：

```assembly
# bar.o
foo:
   # foo 函数的汇编代码
bar:
   # bar 函数的汇编代码

# baz.o
foo:
   # foo 函数的汇编代码
baz:
   # foo 函数的汇编代码
```

编译器将源文件编译为 `.o` 文件后，链接器就会尝试将他们合并为一个可执行文件。这时候它就会发现有一个符号（`foo`）被定义了两次。而它没有能力处理这种问题，所以便报错了。这也就是 C++ 中经常听到的 ODR violation。

但是，有些时候我们确实希望能在头文件中写函数定义，比如一个 header-only 的库。这种情况下我们便可以在函数前加一个 `inline` 关键字。此时连接器就会选择一份定义，然后将其他重复的定义丢掉。



另外值得一说的是在 C++17 之后，`inline` 的语义被推广到了变量上，你可以在全局变量前加上 `inline` 关键字，达到相同的效果。

> 思考一下，如果头文件里不能直接写函数定义，那函数模板呢？没错，C++ 的模板是 implicit inline 的！

## 总结

* `inline` 关键字会作为一个 hint 帮助编译器中端的 Inliner 去分析是否要内联一个函数，减少函数调用产生的开销。
* `inline` 可以让一个函数或者一个变量（C++17之后）可以在多个 Translation unit 有重复的定义，一般用在 header-only 的库中。
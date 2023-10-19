---
title: "链接与库"
date: 2023-03-03T16:32:13+08:00
---

## 虚拟内存

我们首先要对平时所说的内存要有一个清楚的认知，那就是我们所谈的实际上都是虚拟内存，不是操纵真实的物理内存。在没有安装操作系统的裸机上，我们所面对的内存就是物理内存。而随着计算机的发展，一个称为“操作系统”的程序被安装到计算机上，它接管了所有硬件资源，并向我们提供了一个个抽象层。我们作为软件开发人员，只需要与操作系统的抽象层接触，这简化了很多由于硬件不一致带来的麻烦。

而虚拟内存就是操作系统对于实际的物理内存向我们提供的抽象层。现代操作系统多是多任务处理系统，而这一个个“任务”，一般称为进程，都有自己独立的进程地址空间。换句话说，每个进程看待虚拟内存都会认为自己拥有所有的空间（当然它会认为低地址的一部分归内核管），也就是整个系统上好像只有它一个进程。如果他要和其它进程通信，可以使用套接字（socket）等方式。说到这里可以领会到操作系统和计算机网络设计的巧妙性。套接字被设计为两个进程通信的方式，不管是同一台机器上的两个进程还是一个网络中的两个机器的两个进程，都可以用这同一抽象的概念进行通讯，屏蔽了大量具体细节。

## C 和 C++ 的编译模型

这里主要讨论 C 语言的情况，C++ 沿用了 C 语言的模型，没有大的差别。

对于 C 编译器来说，它可以独立编译一个单独的 `.c` 文件，产物为 `.o` 文件。最后我们通过链接器将他们链接成一个可执行文件。

而这中间就产生了一个问题，如果每个 `.c` 文件都是单独编译的，那它又怎么知道其他文件的信息呢？比如说 A 文件中的一个函数需要调用 B 文件中的另一个函数。

举个具体的例子：
```c
// foo.c
int foo(int i) {
    return i + 42;
}
```
```c
// bar.c
int bar(int i) {
    return foo(i) + 24;
}
```
这里 `bar.c` 中的一个函数引用了 `foo.c` 中的一个函数，那我们又如何告诉编译器有关函数 `foo` 的信息呢？最终的设计就是**假装我们有这个函数**。
```c
// bar.c
int foo(int i); // 声明函数 foo，告诉编译器我们有它。

int bar(int i) {
    return foo(i) + 24;
}
```
而随着这些声明的信息越多，我们可以将所有的这些信息放到一个单独的文件中，再 `include` 进来，形成了**头文件**的形式。
```c
// bar.c
#include "foo.h"
int bar(int i) {
    return foo(i) + 24;
}
```

## ELF 文件格式

源文件被编译器编译过后便形成了 object file，它是一个二进制文件，只有0和1。不同操作系统的 Object file 有不同格式，下面只会讨论 Linux 平台下使用的 ELF 格式。

我们将一个 `.c` 文件编译后形成的 Object file 中至少需要存在两部分信息：

* 代码，即代表指令的二进制序列。
* 数据，即代表全局数据的二进制序列。

在设计上，我们将具有相同属性的信息放在一起，称为一个段。而因为 ELF 文件中的数据都是二进制，所以是没办法直接区分的。为了解决这一点，我们又单独分出一个区域叫做段表，记录每个段的名称与它所占区域的映射。而为了能找到段表，我们又把一个 ELF 文件开始一部分固定长度的区域作为头部，其中每个偏移量代表的含义是固定的。利用这点可以把段表的开始位置记录在头部，于是我们便能够得到所有段的相关信息。

下面是一些比较重要的段：

|  段名   |                  作用                   |
| :-----: | :-------------------------------------: |
|  .text  |                  代码                   |
|  .data  |    已初始化的全局变量和局部静态变量     |
|  .bss   |    未初始化的全局变量和局部静态变量     |
| .rodata | 只读数据，如字符串常量和全局 const 变量 |

我们可以通过 objdump 和 readelf 工具查看相关信息。举个例子：

```c
// demo.c
int foo();

int global1 = 42;
int global2;
const int global3 = 43;
int main() {
    static int static1 = 43;
    static int static2;
    return 12;
}
```
用以下命令编译：
```
gcc -c demo.c
```
查看文件头：
```
readelf -h demo.o

ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          640 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         13
  Section header string table index: 12
```

查看各个段的相关信息：
```
objdump -h demo.o

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000000f  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000008  0000000000000000  0000000000000000  00000050  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000008  0000000000000000  0000000000000000  00000058  2**2
                  ALLOC
  3 .rodata       00000004  0000000000000000  0000000000000000  00000058  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      0000002c  0000000000000000  0000000000000000  0000005c  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  00000088  2**0
                  CONTENTS, READONLY
  6 .note.gnu.property 00000020  0000000000000000  0000000000000000  00000088  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .eh_frame     00000038  0000000000000000  0000000000000000  000000a8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
```

查看代码段：
```
objdump -s -d demo.o

Contents of section .text:
 0000 f30f1efa 554889e5 b80c0000 005dc3    ....UH.......].
Contents of section .data:
 0000 2a000000 2b000000                    *...+...
Contents of section .rodata:
 0000 2b000000                             +...
Contents of section .comment:
 0000 00474343 3a202855 62756e74 75203131  .GCC: (Ubuntu 11
 0010 2e332e30 2d317562 756e7475 317e3232  .3.0-1ubuntu1~22
 0020 2e303429 2031312e 332e3000           .04) 11.3.0.
Contents of section .note.gnu.property:
 0000 04000000 10000000 05000000 474e5500  ............GNU.
 0010 020000c0 04000000 03000000 00000000  ................
Contents of section .eh_frame:
 0000 14000000 00000000 017a5200 01781001  .........zR..x..
 0010 1b0c0708 90010000 1c000000 1c000000  ................
 0020 00000000 0f000000 00450e10 8602430d  .........E....C.
 0030 06460c07 08000000                    .F......

Disassembly of section .text:

0000000000000000 <main>:
   0:   f3 0f 1e fa             endbr64
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   b8 0c 00 00 00          mov    $0xc,%eax ; 0xc 就是12，即返回12
   d:   5d                      pop    %rbp
   e:   c3                      ret
```

## 链接与符号

链接的本质就是将一个个 Object file 拼凑在一起。如果说每个 Object file 中变量和函数等都是独立的，那么这个过程非常简单。但是，实际过程中往往是这个文件引用了另一个文件的函数，我们必须解决这个问题，如果 `foo.c` 中的函数调用了一个外部函数 `bar()`，我们生成的 `jmp` 指令一定要指向正确的地址。

所以连接器的主要职责之一就是确定每个符号实际的地址。为此，每一个 ELF 文件都维护了一个符号表，用于表示所有符号的名称和它相关信息（地址，类型等）的映射。

举个例子：
```c
int i = 42;
int foo(int);
void bar() { foo(i); }
```
我们检查它的所有符号：
```
nm demo.o
0000000000000000 T bar
                 U foo
0000000000000000 D i
```
`T` 代表代码段已定义的符号，`D` 代表数据段已定义的符号，`U` 代表未定义的符号，需要后期链接器处理。

### C++ Name Mangling

因为 C++ 中有函数重载的存在，所以多个函数可以重名。如果不处理，那么便会出现符号冲突的情况。对此，C++ 编译器会将函数参数也作为表示加进符号名称中。不同的 ABI 有不同的规则，对于 Linux 使用的 Itanium ABI，将上面的例子作为 C++ 代码编译后符号情况如下：
```
nm demo.o

0000000000000000 D i
0000000000000000 T _Z3barv
                 U _Z3fooi
```

详细规则可以查阅 Itanium ABI 文档，这里只要知道 name mangling 的存在即可。而为了使 C 代码能够正确地链接 C++ 代码，C++ 也提供了不 mangle 符号的办法：
```c++
extern "C" {
    int foo();
}
```
### 弱符号和强符号

符号的类型分为强符号（strong symbol）和弱符号（weak symbol）。如果有两个符号同名，取强符号的定义，多个同名强符号会冲突报错，多个同名弱符号是未定义行为。

编译器默认函数和初始化了的全局变量为强符号，未初始化的全局变量为弱符号。GCC 也提供了一个拓展，定义任何一个符号为弱符号：
```c
__attribute__((weak)) weak = 42;
```

弱符号对于库十分有用，库定义的弱符号可以被用户定义的强符号覆盖，使得程序使用自定义的库函数。

比如 libcxx 中 operator new 的定义：
```cpp
// Implement all new and delete operators as weak definitions
// in this shared library, so that they can be overridden by programs
// that define non-weak copies of the functions.

_LIBCPP_WEAK
void *
operator new(std::size_t size) _THROW_BAD_ALLOC
{
    ...
}
```
用户可以提供自己的 operator new 函数，覆盖自带的库函数。

## 静态链接
一个静态库可以看作一组 Object file 的集合。可以用以下命令创建一个静态库：
```
gcc -c foo.c
gcc -c bar.c
ar -rc libtest.a foo.o bar.o
```

其实 `ar` 本身就是将多个文件打包在一起的工具，没什么稀奇的。

将静态库与其他 object file 链接成可执行文件的细节属于连接器内部实现，这里不讨论太多，但总体思想就是将 静态库（`.a`）中所有引用到的 `.o` 文件全部合并在一起，这也是为什么静态链接的程序体积很大的原因。

### C++ 中的全局对象
我们通常认为 C 和 C++ 程序是从 `main` 开始，实际上为了程序能够正常运行，在此之前我们还需要初始化进程环境，调用全局对象构造函数等很多事情。为了实现全局对象的构造和析构，ELF 还定义了两个特殊的段：

* `.init` 保存着进程的初始化代码，及全局对象的构造函数调用。
* `.fini` 保存着进程的终止代码，及全局对象的析构函数调用。

## 动态链接
静态链接的思路很简单也很有效，但它的最大问题就是体积太大。为此，动态链接应运而生。

动态链接的基本思想就是链接这个构成从本来的程序加载前推后到了装载时。

### 位置无关代码
共享库的一个主要目标就是允许多个进程使用相同的库代码，从而节省硬盘和内存。一个简单的思路是预先分配一个专用的地址空间，总是将共享库加载到这一位置。然而，这对地址空间的使用率不高，还容易造成大量内存碎片。为了解决这些问题，现代操作系使用了一种称为位置无关代码（Position Dependent Code）的技巧。

#### 数据引用

编译器会在数据段开始的地方创建一个表，称为全局偏移量表（Global Offset Table）。这个表是一个数组，每个表项是引用的全局数据目标（函数或者全局变量）的地址。又因为数据段和代码段之间的距离总是一个常量，我们可以通过指令指针（%rip）和一个偏移量找到对应的表项。

举个例子：
```asm
foo:
    mov 0x2008b9(%rip), %rax ; 找到 GOT 中对应表项，注意是地址。
    addl $0x1, (%rax) ; 因为是地址，所以要先解引用，再操作。
```
等价的 C 代码为：
```c
%rax = GOT[3]; // GOT[3] 保存的就是该全局变量实际的地址。
*rax += 1;
```

#### 代码引用

假设程序调用了一个由共享库定义的函数，编译器无法预测这个函数实际的地址，因为它可以被加载到任意位置。为了解决这个问题，我们使用了一种称为 Lazy Binding 的技术，将地址的绑定推迟到第一次调用的时候。

Lazy Binding 通过全局偏移量表（Global Offset Table）和过程链接表（Procedurev Linkage Table）共同实现。过程链接表 PLT 也是一个数组，它像一个代理，它的表项负责调用实际被调用的函数。GOT 正如上面所提到的那样，保存着被调用函数的实际地址（注意它需要在运行时被解析）。

举一个例子：
1. 当用户程序第一次调用被定义在共享库中的函数 `foo` 时，会进入对应的 `PLT` 表项中。该 `PLT` 表项是一个 `jmp` 指令，会跳转到对应 `GOT` 表项中所定义的地址。
2. 因为这是第一次调用，`GOT` 表项并不是函数 `foo` 真正的地址，而是之前 `jmp` 指令的下一个地址。在这个地址上的指令是一条 `push` 指令，代表函数 `foo` 的 ID。接着往下执行，又是一条 `jmp` 指令，这次会跳转到 PLT[0] 上，也就是调用动态连接器。
3. 动态连接器找到函数 `foo` 的实际位置，重写其对应的 `GOT` 表项。
4. 自此，以后所有调用经过步骤1便可跳转到对应位置。

### 符号可见性

按照原本的链接模型，共享库中所有的符号都应该是外部公开可以被访问的。但这个设计明显存在很多问题，丢失了良好的封装性只是一方面，最重要的是，当符号的数量非常大时，比如一些滥用模板的 C++ 库（没错我说的就是 boost），会导致加载时间大大增加。

编译器为此提供了一个命令行参数，可以把所有的符号都外部默认不可见。
```
-fvisibility=hidden
```

接着我们可以用一个 GCC 提供的拓展来标记我们需要导出的符号：
```cpp
void __attribute__((visibility("default"))) Exported()
{
    // ...
}
```
如此便做到了减小动态库体积（减小符号表）和降低符号碰撞。

## 引用资料
* [1] 深入理解计算机系统
* [2] 程序员的自我修养 -- 链接，加载与库
* [3] [gcc.gnu.org/wiki/Visibility](https://gcc.gnu.org/wiki/Visibility)

---
title: "CUDA初学笔记"
author: "Jun"
date: 2022-11-22T17:03:28+08:00
---

```cpp
__global__ void Kernel(float* A, int N) {
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    if (x < N)
      A[x] = 1;
}
int main() {
    // ...
    Kernel<<<2,32>>>(A, 64);
}
```
上面是一个简单的 CUDA 例子，其中初始化了一个长度为64的单精度浮点数数组。注意到我们在`Kernel`函数前有一个 `__global__` 修饰，这表示这个函数为“核函数”，将被 CUDA 编译器编译，而我们正常的代码，仍将使用 C/C++ 编译器编译。

## CUDA 限定词
下面是 CUDA 提供的所有用来说明代码的种类的限定词：

* `__global__` 核函数，返回类型必须为 `void`。从 host 端被调用，在 device 端运行。形式为 `Kernel<<<numOfBlocks, threadPerBlock>>>(...)`。**注意这是个异步调用。**
* `__device__` 用来修饰在 device 端编译运行的代码，返回类型没有限制。
* `__host__` 用来修饰在 host 端编译运行的代码，一般可以直接省略。如果一个函数既可以在 host 端，也可以在 device 端运行，则可以使用 `__host__` 和 `__device__` 同时修饰它。**这时候相当于将这份代码编译两份。**

> host 和 device 都是 CUDA 中的特有名词，可以理解为 host 就是 CPU，而 device 是 GPU。

## 网格，块和线程

我们每调用一次核函数都会启动一个网格，一个网格(grid)中有若干个块(block)，block 中有含有若干个线程(thread)。 

以上面的的代码举例：
```cpp
Kernel<<<2,32>>>(A, 64);
```
`Kernel` 是核函数，上面我们说到我们可以通过调用它在 CPU 端与 GPU 沟通。而这三个尖括号，则是 CUDA 提供的扩展，我们可以通过它指定启用的网格中有多少个 block, 而每个 block 中又含有多少 thread。

需要注意的是，三个尖括号中传递的两个元素的类型并不是普通的整数，而是 `dim3` 类型。它被定义在了 `vector_types.h` 中。可以将它看作是下面这个样子：
```cpp
struct dim3 {
    unsigned int x = 1;
    unsigned int y = 1;
    unsigned int z = 1;
};
```
也就是说，我们指定的 block 数和 thread 数可以是一维，二维或三维的！而当我们像上面那样使用一个数字的时候，则隐式转换成一个一维的 `dim3`。
```cpp
dim3 d1(2,1,1);
dim3 d2(2,3,1);
dim3 d3(2,3,4);

Kernel<<<d1,d1>>>(...); // 使用了2个 block, 每个 block 中有2个 thread。
Kernel<<<d2,d2>>>(...); // 使用了2x3个 block, 每个 block 中有2x3个 thread。
Kernel<<<d3,d3>>>(...); // 使用了2x3x4个 block, 每个 block 中有2x3x4个 thread。
```
在上面的第一个调用中，我们可以想象 CUDA 启动了这样的一个 grid:
```
+-------------------------------------------------------------------+
| +------------------------------+ +------------------------------+ |
| | +------------++------------+ | | +------------++------------+ | |
| | |  thread 0  ||  thread 1  | | | |  thread 0  ||  thread 1  | | |
| | +------------++------------+ | | +------------++------------+ | |
| |            block 0           | |            block 1           | |
| +------------------------------+ +------------------------------+ |
+-------------------------------------------------------------------+
```
在上面的第二个调用中，我们可以想象 CUDA 启动了这样的一个 grid:
```
+-------------------------------------------------------------------+
| +--------------------++--------------------++--------------------+|
| | +----++----++----+ || +----++----++----+ || +----++----++----+ ||
| | +----++----++----+ || +----++----++----+ || +----++----++----+ ||
| | +----++----++----+ || +----++----++----+ || +----++----++----+ ||
| +--------------------++--------------------++--------------------+|
| | +----++----++----+ || +----++----++----+ || +----++----++----+ ||
| | +----++----++----+ || +----++----++----+ || +----++----++----+ ||
| | +----++----++----+ || +----++----++----+ || +----++----++----+ ||
| +--------------------++--------------------++--------------------+|
+-------------------------------------------------------------------+
```
三维的就不画了，太复杂。

而为什么我们需要这种2维甚至3维的表示呢？这主要是因为 GPU 最初都是处理图形学的问题，而图形学中难免会有这种多维的结构。这个时候，在语言中有着原生的多维表示方法就显得比较方便。

需要注意的是，以上只是 CUDA 的心理模型，不是实际的硬件实现。但是，这个心理模型可以很好地映射到它的硬件实现上，这对我们理解 CUDA 非常重要。

CUDA 提供了4个内置变量用于访问当前线程在 gird 中的信息，他们都是 `dim3` 类型：
* `threadIdx` 线程在 block 中的索引，从0开始。
* `blockDim` 一个 block 中线程的数量。
* `blockIdx` 线程的 block 在 grid 中的索引，从0开始。
* `gridDim` block 的总数量。

在最开始的例子中，我们使用了这样一段代码来访问线程在 grid 中的总索引：
```cpp
int x = blockIdx.x * blockDim.x + threadIdx.x;
```
这个公式其实很好理解，可以把它看作 C 语言中的二维数组的访问：
```
index = BlockIndex * BlockSize + ThreadIndex
```

## 共享内存
每个单独的线程拥有一组寄存器和局部内存可供自身使用。而全局内存则提供了较慢的，可供所有线程使用的空间。除此之外，**每个 block 中还有一份共享内存，可供该 block 中的所有线程共享使用。**

我们可以用以下方式声明一段共享内存：
```cpp
__shared__ float shared_mem[64];
```
共享内存是一段相较于全局内存和局部内存更快的内存，所以我们可以使用它作为一个缓冲区。举个例子，我们可以通过先将部分全局内存中的内容复制到共享内存上，然后处理后写回的方式大大提升 CUDA 程序的性能。

## 硬件模型
写 CUDA 不能不懂英伟达 GPU 的硬件模型，很多优化实际上都是根据其硬件上的特点在编程上做出的一些调整。

上面我们谈到网格，块和线程的心理模型，而事实上，一个 GPU 由多个流处理器簇 (Streaming Multiprocessor), 简称为 SM 组成。每个 SM 中又含有多个流处理器 (Streaming Processor), 简称为 SP, 或者叫 CUDA core。可以把他们理解成下面这样的映射：
```
SM => block
SP => thread
```
一个 SM 上可以并行地运行一个或多个 block, 这取决于硬件性能。而当一个 block 运行在一个 SM 时，必须完全执行到退出，不能被迁移到其他 SM 上（调试模式除外）。

而在一个 SM 中的所有线程，又是以 `wrap` (线程束) 的形式进行调度的。`wrap` 就是一组线程，通常的大小为32。所以一个 block 中会有 `blockSize/warpSize` 个 `wrap`。如果 block 中的线程数不能被 `wrapSize` 整除，那么将向上取整。也就是说会有一个 wrap 中将有若干个 thread 是无效的，不做任何工作。这显然不是我们所希望的，**所以 block 的大小，也就是 `blockDim`，通常情况应取32的倍数，才不会造成性能的浪费。** 

另外，`blockDim` 也不能过大，因为一个 block 中的所有线程共用一个寄存器仓库 (register file)，如果线程过大就会导致寄存器不够用使部分暂时没有被使用的寄存器被 spill 到所在 SM 的一级缓存中，导致性能的降低。

一个 `wrap` 中的32个线程以`SIMT (Single Instruction Multiple Thread)` 的方式执行。也就是说一个 `wrap` 中的所有线程在同一时间内会执行相同的一行代码。

这其实会造成一个潜在的问题，举个例子：
```cpp
int tid = blockDim.x * blockIdx.x + threadIdx.x;
if (tid % 2 == 0) {
    ...
}
```
由于一个 `wrap` 中的每个线程的线程号是不一样的，当执行到这个 `if` 语句时，有些线程必定会不满足这个条件，无法执行下去。这就导致了这个 `wrap` 中的所有线程将无法执行同一行代码。这种情况被称为 `warp divergence`。**当 `warp divergence` 发生时，无法进入分支语句的线程就会进入等待状态，造成闲置浪费资源。**

## Bank conflicts

为了获取高带宽，共享内存在物理上被分成32个（一个 `wrap`）大小相同的 bank，然后再将这32块小空间并联。（据说在最新架构成32变成了16，也就是半个 `wrap`）

也就是说，`shared_mem[0]` 的数据存放在 `bank[0]` 处，`shared_mem[31]` 的数据存放在 `bank[31]` 处，然后 `shared_mem[32]` 的数据也开始存放在 `bank[0]` 处。

如下图所示：
```
+----+     +----+ +----+          
|    | ... |  32| | 0  |  bank 0  
+----+     +----+ +----+          
+----+     +----+ +----+          
|    | ... |  33| | 1  |  bank 1  
+----+     +----+ +----+          
       ...                ...        
                                  
+----+     +----+ +----+          
|    | ... | 63 | | 31 |  bank 31  
+----+     +----+ +----+     
```
bank 与 bank 之间并行读取和写入不影响，但单个 bank 的访问则是串行的。也就是说**当一个或半个 `wrap` 中多个线程访问了同一个 bank 时，线程只能一个一个的读取和写入，造成 bank conflicts。**

注意一个特例，如果一个或半个 `wrap` 中的所有线程都访问了一个地址，则 GPU 会开启广播 (broadcast) 机制，也不会发生冲突。

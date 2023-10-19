---
title: "CMake学习笔记"
author: "Jun"
date: 2021-06-19T13:24:06+08:00
---

## CMake简介

cmake是一个跨平台的构建系统。众所周知，像C++这样的语言构建过程非常痛苦，各个不同的平台使用的工具也各不相同。在Linux系统上我们一般使用Makefile，在Windows和Mac OS X 上也有各家IDE的配置文件。cmake则可以实现编写一次，在各个平台上生成所需的编译配置文件，最后我们可以再用生成的文件构建即可。可以把cmake理解为一个中间层，对于复杂多样的构建过程提供了统一的接口。

下面是我学习cmake的笔记和思考。以下均在Linux下操作，适用于所有Unix-like的操作系统。

## 资料

- [官网](https://cmake.org/cmake/help/v3.20/)
- [文档](https://cmake.org/cmake/help/v3.20/)
- [cmake-examples](https://github.com/ttroy50/cmake-examples)


## 基本操作

cmake的配置文件名一般为`CMakeLists.txt`，至少存在于项目的根目录，对于复杂的项目，子项目中也会有。

因为cmake会生成一系列配置文件，为了不污染项目，我们一般采用外部构建的办法，具体操作如下：

```
project
├── CMakeLists.txt
├── include
│   └── source.h
└── src
    └── source.cc
```

假如有一个结构如上的项目，我们一般在项目根目录下再手动创建一个目录为`build`，进入此目录后`cmake ..`，将配置文件写入`build`目录。

```bash
mkdir build
cd build
cmake ..
make
```

## 通用选项

### 限定cmake最低版本

```cmake
cmake_minimum_required(VERSION 3.5)
```
### 设置项目的名称

```cmake
project(hello)
```
### 设置编译器

```cmake
set(CMAKE_C_COMPILER "/usr/bin/clang") # C++ 编译器
set(CMAKE_CXX_COMPILER "/usr/bin/clang++") # C++ 编译器
```

也可以在调用 cmake 命令时使用：
```bash
cmake -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ ../
```
### 设置C++标准

```cmake
set(CMAKE_CXX_STANDARD 17)
```
## 添加源文件

`SET()`可以用来创建一个变量，第一个参数为参数名，剩下的参数则为值。我们一般用它创建源文件的别名，这样就可以用`${变量名}`指代所有源文件了。

```cmake
SET(SOURCES
  src/source1.cc
  src/source2.cc
  ...
)
```
## 添加可执行文件
```cmake
add_executable(executable_name sources_files)
```
我们用`add_executable()`添加一个可执行文件，第一个参数是可执行文件的名字，剩下的参数可以是指定的文件名，也可以是之前创建的`SOURCES`变量

```
#直接指定文件名
add_executable(demo
  src/source1.cc
  src/source2.cc
)
#用变量名
add_executable(demo ${SOURCES})
```

## 添加库 
我们用`add_library()`添加一个库，第一个参数为添加的库名，第二个参数表明为是static(静态库)，或者是shared(动态库)，剩下的参数则是源文件，当然与上面一样，可以是指定的文件名，或者是用`SET()`创建的变量
```cmake
add_library(lib STATIC src/lib.cpp)
add_library(lib SHARED src/lib.cpp)
```
## 设置头文件路径
我们用`target_include_directories()`指定头文件的路径，第一个参数可以为此前添加的可执行文件名，或者为此前添加的库名。注意我们应该在添加完target之后再使用此方法，否则会提示错误。因为我们这里include的第一个参数是指定需要添加头文件的target。
```cmake
target_include_directories(
  target_name
  PUBLIC
  ${PROJECT_SOURCE_DIR}/include
)
```
## 链接库

```cmake
target_link_libraries(
  target_name
  PRIVATE
  lib_name
)
```

## 安装
cmake提供了命令行参数，使我们可以指定安装的前缀

```bash
cmake .. -DCMAKE_INSTALL_PREFIX=/install/location
```

我们也可以在配置中指定

```cmake
install(TARGETS
  target_name
  DESTINATION
  bin
)
```

```cmake
install (TARGETS 
  target_name
  LIBRARY 
  DESTINATION 
  lib
)
```



## 含有第三方库或子项目

有多个子项目的大型项目，我们可以把各个子项目组合起来，每个子项目都有自己的`CMakeLists.txt`，在根目录的`CMakeLists.txt`使用`add_subdirectory()` 。

```
add_subdirectory(sub_project_name)
```

其中`sub_project_name`为子项目的名称。

## 单元测试

cmake已经包含一个一个工具[CTest](https://cmake.org/Wiki/CMake/Testing_With_CTest)，我们可以通过它`make test`从而进行单元测试。至于单元测试的框架，我们可以使用[googletest](https://github.com/google/googletest)。

具体过程如下：

- 先将`gooletest`作为一个子项目用`add_subdirectory()`添加进来
- `enable_testing() ` 开启CTest，使用单元测试
- `target_link_libraries(target GTest::GTest GTest::Main)` 将googletest链接进target
- `add_test(test_all target)`添加测试

## 例子
```
├── build
├── CMakeLists.txt
├── include
│   ├── a.h
│   └── b.h
├── src
│   ├── a.cc
│   ├── b.cc
│   └── unittest.cc
└── third-party
    └── google-test
        ├── CMakeLists.txt
        └── CMakeLists.txt.in
```
对于像上面这样的一个小项目，我们编译了一个静态库（lib)，并使用googletest作为单元测试框架。它的`MakeLists.txt`如下:
```cmake
# 指定最低版本
cmake_minimum_required(VERSION 3.5)
# 指定编译器
set(CMAKE_CXX_COMPILER "/usr/bin/clang++")
# 指定C++标准
set(CMAKE_CXX_STANDARD 17)
# 指定项目名
project(test)
# 导入子项目googletest
add_subdirectory(third-party/google-test)
# 添加静态库
add_library(lib
  STATIC 
  src/a.cc
  src/b.cc
)
# 添加头文件目录
target_include_directories(
  lib
  PUBLIC
  ${PROJECT_SOURCE_DIR}/include
)
# 开启单元测试
enable_testing()
# 添加可执行文件
add_executable(
  main
  src/unittest.cc
)
# 链接静态库与第三方库
target_link_libraries(
  main
  PRIVATE
  test
  GTest::GTest 
  GTest::Main
)
# 启动测试
add_test(test_all main)
```

---
title: "如何编译 GraalVM"
author: "Jun"
date: 2023-07-27T23:45:13+08:00
---

下面记录了我如何在 Linux (Ubuntu22.04 LTS) 上编译 GraalVM 的步骤。

```bash
mkdir lava # 创建一个工作区
git clone https://github.com/graalvm/mx.git # 下载编译用的工具
git clone https://github.com/graalvm/graal.git # 下载源代码

export PATH=$(pwd)/mx:$PATH # 将 mx 加到环境变量里
mx fetch-jdk # 安装 JDK，这里我选择的是 jdk17，下载完后它会提示你设置 JAVA_HOME 环境变量
export JAVA_HOME=/home/jun/.mx/jdks/labsjdk-ce-17-jvmci-23.1-b02 # 根据上一步的提示设置
export PATH=$JAVA_HOME/bin:$PATH # 将 JDK 加到环境变量中

cd graal/compiler
mx build # 编译
mx intellijinit # 生成 IDEA 的配置文件，有了这个就可以在 IDEA 中打开了
```

在 IDEA 中打开 `graal/compiler` 后会显示没设置 JDK，手动指向 `~/.mx/jdks` 里对应的就可以了。
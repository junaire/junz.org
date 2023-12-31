---
title: "浅谈HTTPS证书"
author: "Jun"
date: 2021-03-01T12:37:06+08:00
---
## HTTPS和SSL/TLS协议

### 背景

传统的http协议是明文传输的，可能遭到篡改和监控，所以引入了https协议。

### 原理

HTTPS实际上是"HTTPS over SSL"。如果原来的 HTTP 是塑料水管，容易被戳破；那么如今新设计的 HTTPS 就像是在原有的塑料水管之外，再包一层金属水管。一来，原有的塑料水管照样运行；二来，用金属加固了之后，不容易被戳破。这样做的好处是提高了兼容性和可拓展性。

### 实现

在进行通信之前，双方交换公钥，并使用自己的私钥进行解密。

服务端随机生成一个密钥对，将私钥留存，并将公钥发送给客户端。客户端收到公钥后，将自己的密钥用服务端的公钥加密并发送给服务端，服务端收到密文后，用自己的私钥进行解密，得到用于通信的密钥。

### 难点

在服务端与客户端进行密钥交换时，如果有恶意攻击者在通讯线路上劫持数据并替换成自己的密钥对，那么安全性将得不到保证，这种攻击类型称之为**中间人攻击**或者**Man-In-The-Middle attack**

### 解决办法

在客户端收到服务端的公钥时，进行核对。

引入一个第三方，即certificate authority，简称CA。CA会使用自己的私钥对服务端的公钥和一系列信息进行加密，得到数字证书（Digital Certificate）。接着服务端将此数字证书与公钥一同发给客户端，客户端则会验证此CA证书是否合法（浏览器内置了大量主流权威CA证书的公钥）于是客户端便可以验证公钥的真实性。




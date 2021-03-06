### 高性能MySQL 第三版

[TOC]

### 第1章 MySQL架构与历史

#### 1.1 MyQL逻辑架构

> 客户端 --> 链接/线程处理 --> 查询缓存/解析器 --> 优化器 --> 接口对接存储引擎

MySQL服务器是一个三层结构，最上层的服务并不是MySQL所独有的，大多数基于网络的C/S的工具或者服务器都有类似的架构。比如链接处理、授权认证、安全等等。

第二层架构是MySQL比较有意思的部分。大多数MySQL的核心服务都在这一层，包括查询解析、分析、优化、缓存以及所有的内置函数（例如，日期、时间、数学和加密函数），所有跨存储引擎的功能都在这一层时间：存储过程、触发器、视图等。

第三层包含了存储引擎。存储引擎负责MySQL中数据的存储和提取。每个引擎都有它的优势和劣势。

服务器通过API与存储引擎的进行通信，这些接口屏蔽了不同存储引擎之间的差异，使得这些差异对上层的查询过程透明。存储引擎API包含几十个底层函数，用于执行诸如“开始一个事物”或者“根据逐渐提取一行记录”等操作。

**但存储引擎不会去解析SQL（InnoDB是一个例外，它会解析外键定义，因为MySQL服务器本身没有实现该功能）**

不同存储引擎之间也不会互相通信，而只是简单地相应上层服务的请求。

##### 1.1.1 链接管理与安全性

MySQL每个客户端链接都会在服务器进程中拥有一个线程，这个链接的查询只会在这个单独的线程中执行，该线程只能轮流在某个CPU核心或者CPU中运行。

服务器会负责缓存线程，因此不需要为每一个新建的链接创建或者销毁线程。（MySQL5.5及以上提供了一个API，支持线程池（Thread-Pooling）插件，可以使用池中少量的线程来服务大量的链接）

当Client连接到MySQL服务器时，服务器需要对其进行认证。认证基于用户名、原始主机信息和密码。

> MySQL中一个账号包含用户名和连接发起时所在的Host。认证时不仅要验证用户名，也要验证Host。可以理解为用户名+Host组成了一个认证账户。同一个用户在不同主机可以设置不同密码。

如果使用了安全套接字（SSL）的方式连接，还可以使用X.509证书认证。一旦客户端连接陈宫，服务器会继续验证客户端是否具有执行某个特定查询的权限。

##### 1.1.2 优化与执行

MySQL会解析查询，并创建内部数据结构（解析树），然后对其进行各种优化，包括重写查询、决定表的读取顺序，以及选择合适的索引等。

用户可以通过特殊的关键字提示（hint）优化器，影响它的决策过程。也可以请求优化器解释（explain）优化过程的各个因素，使用户可以知道服务器是如何进行优化决策的。
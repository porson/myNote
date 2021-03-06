## Go语言并发编程读书笔记
[TOC]

---

### 前言
> Go语言这些年来的发展变化

#### Go语言完成自举
-   运行时系统改进
    - 更高效的调度器、内存管理、GC

#### 标准工具的增强
- go generate
- go tool trace
- go tool compile
- go tool asm
- go tool link

#### 访问控制细化
> 除了原先的两种访问控制级别（公开和包级私有）之外，又多了一种——模块级私有。这是通过把名称首字母大写的程序实体放入internal代码包实现的。

#### vendor机制的支持
> 用于存放其父目录中的代码包所依赖的那些代码包。
> 在程序被编译时，编译器会有限引用存于其中的代码包。这位固化程序的依赖代码迈出很重要的一步。
> 在Go1.7中，vendor目录以及背后的机制被正式支持。

---

### 第一章 初识Go语言
#### 1.1 语言特性
-   开放源代码

-   静态类型和编译型
    - 每个变量或常量必须在声明时制定类型，且不可改变
    - 程序必须通过编译生成归档文件或可执行文件后才能被使用或执行

-   跨平台

-   自动垃圾回收（GC）
    - 允许我们对此项工作进行干预

-   原生的并发编程

    -   拥有自己的并发编程模型，goroutine + channel，关键字 go
>进程线程与协程的区别：
> 	进程与线程是确切存在的，一个进程可以拥有多个线程，他们确切占用CPU时间片进行运算操作，每个线程对CPU的占用是不可预测的。
> 	协程（例程）是由语言层面实现的，由编译器控制某些代码块出入栈，对CPU时间的占用是可预测的。协程是人工实现的更小的线程，或线程的线程。

- 完善的构建工具
- 多编程范式，支持函数式编程，支持面向对象编程，由借口类型与实现类型的概念，用嵌入代替了继承。
- 代码风格强制统一（go fmt）
- 高效的编程和运行。
- 丰富的标准库。

#### 1.2 安装和设置

##### Go语言包结构

- api文件夹：用于存放依照Go版本顺序的API增量列表文件。这里说的API包含公开的变量、常量、函数等。作用为Go语言API检查。

- bin文件夹：用于存放主要的标准命令文件，包括go、godoc和gofmt

- blog文件夹：用于存放官方博客中的所有文章，这些文章都是MarkDown格式的。

- doc文件夹：用于存放标准库的 HTML格式的程序文档。可以利用godoc命令启动一个web程序展现这些文档。

  ```shell
  godoc -http=:port
  ```

- lib文件夹：用于存放一些特殊的库文件。

- misc文件夹：用于存放一些辅助类的说明和工具。

- pkg文件夹：用于存放安装Go标准库后的所有归档文件。

  - 其中有名称为*linux_amd64*的文件夹，我们成为平台相关目录。
  - 通过go install命令，Go程序（这里指标准库中的程序）会被编译成平台相关的归档文件并存放到其中。
  - *pkg/tool/linux_amd64*文件夹存放了使用Go制作软件是用到的很多强大命令和工具。

- src文件夹：用于存放Go自身、Go标准工具以及标准库的所有源码文件。

- test文件夹：存放用来测试和验证Go本身的相关文件。
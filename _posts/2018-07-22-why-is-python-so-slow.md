---
layout: post
title: "为什么 Python 这么慢（译）"
description: "Java如何在速度方面与C或C ++或C＃或Python进行比较？ 答案很大程度上取决于您运行的应用程序类型。 没有基准是完美的，但计算机语言基准游戏是一个很好的起点。"
categories: [python]
tags: [python, performance]
redirect_from:
  - /2018/07/22/
---

[原文链接](https://hackernoon.com/why-is-python-so-slow-e5074b6fe55b) 

  > 此译文仅供参考，这是学习英语的译文，仅供参考。



Python 正在繁荣发展。他用在DevOps，数据科学，Web 开发和安全。
然而，它并没有在速度上取得任何优势。


  > Java如何在速度方面与C或C ++或C＃或Python进行比较？ 答案很大程度上取决于您运行的应用程序类型。 没有基准是完美的，但计算机语言基准游戏是一个很好的起点。

十多年来，我一直在谈论计算机语言基准游戏; 与Java，C＃，Go，JavaScript，C ++等其他语言相比，Python是最慢的。 这包括JIT（C＃，Java）和AOT（C，C ++）编译器，以及JavaScript等解释语言。

注意：当我说“Python”时，我是在谈论参考CPython语言实现。 我将在本文中引用其他运行时。

> 我想回答这个问题：当Python完成一个类似的应用程序比另一种语言慢2-10倍时，为什么它变慢，我们不能让它更快？

以下是最多回答的理论：

* “这是GIL（全局解释器锁）”
* “这是因为它是解释而不是编译的”
* “这是因为它是一种动态类型的语言”

其中哪个一个原因对性能影响最大？

## “这是GIL”
现代计算机配备了具有多个内核的CPU，有时还有多个处理器。 为了利用所有这些额外的处理能力，操作系统定义了一个称为线程的低级结构，其中一个进程（例如Chrome浏览器）可以生成多个线程并为系统内部提供指令。 这样，如果一个进程特别是CPU密集型，则可以跨核心共享该负载，这有效地使大多数应用程序更快地完成任务。

在我写这篇文章时，我的Chrome浏览器有44个线程打开。 请记住，基于POSIX（例如Mac OS和Linux）和Windows OS之间的线程结构和API是不同的。 操作系统还处理线程的调度。

如果您之前没有进行过多线程编程，那么您需要快速熟悉锁定的概念。 与单线程进程不同，您需要确保在更改内存中的变量时，多个线程不会尝试同时访问/更改相同的内存地址。

当CPython创建变量时，它会分配内存，然后计算对该变量的引用数量，这是一个称为引用计数的概念。 如果引用数为0，则从系统中释放该内存。 这就是为什么在for循环的范围内创建一个“临时”变量不会破坏应用程序的内存消耗。

当变量在多个线程内共享时，CPython如何锁定引用计数，就会遇到挑战。 有一个“全局解释器锁”，它小心地控制线程执行。 解释器一次只能执行一个操作，无论它有多少线程。

**这对Python应用程序的性能意味着什么？**
如果您有单线程，单个解释器应用程序。 它对速度没有影响。 删除GIL不会影响代码的性能。

如果您希望通过使用线程在单个解释器（Python进程）中实现并发，并且您的线程是IO密集型的（例如，网络IO或磁盘IO），您将看到GIL竞争的后果。

![ From David Beazley’s GIL visualised post http://dabeaz.blogspot.com/2010/01/python-gil-visualized.html
](/assets/images/why-python-so-slow/gil-visualised.png)
如果您有一个Web应用程序（例如Django）并且您正在使用WSGI，那么对您的Web应用程序的每个请求都是一个单独的Python解释器，因此每个请求只有一个锁。 由于Python解释器启动缓慢，因此一些WSGI实现具有“守护进程模式”，可以随时随地保存Python进程。

**那么其他Python运行时呢？**
[PyPy有一个GIL](http://doc.pypy.org/en/latest/faq.html#does-pypy-have-a-gil-why)，它一般比CPython快3倍。

[Jython没有GIL](http://www.jython.org/jythonbook/en/1.0/Concurrency.html#no-global-interpreter-lock)，因为Jython中的Python线程由Java线程表示，并且受益于JVM内存管理系统。

**JavaScript在这点是如何做的？**
首先所有Javascript引擎都使用标记和扫描垃圾收集。 如上所述，GIL的主要需求是CPython的内存管理算法。
JavaScript没有GIL，但它也是单线程的，所以它不需要一个。 JavaScript的事件循环和Promise / Callback模式是实现异步编程以代替并发的方式。 Python与asyncio事件循环有类似之处。

## “这是因为它是一种解释性的语言”

我听到了这个很多，我发现CPython实际工作方式的可以粗略简化。 如果你在一个终端上编写了python myscript.py，那么CPython将启动一长串的读，解，解析，编译，解释和执行代码。
如果您对该过程的工作方式感兴趣，我之前已经写过：[Modifying the Python language in 6 minutes](https://hackernoon.com/modifying-the-python-language-in-7-minutes-b94b0a99ce14)

该过程的一个重点是创建.pyc文件，在编译阶段，字节码序列被写入Python 3中__pycache __ /内的文件或Python 2中的同一目录中。这不仅适用于您的脚本，还包含您导入的所有代码，第三方模块。
所以大部分时间（除非你编写的代码只运行一次？），Python正在解释字节码并在本地执行它。 与Java和C＃.NET相比：
> Java编译为“中间语言”，Java虚拟机读取字节码，并及时将其编译为机器代码。 .NET CIL也是同样的，.NET公共语言 - 运行时，CLR，使用即时编译到机器代码。
那么，如果Python使用虚拟机和某种字节码，为什么Python在基准测试中比Java和C＃慢得多？ 首先，.NET和Java是JIT编译的。
JIT或即时编译需要一种中间语言，以允许将代码拆分为块（或帧）。 提前（AOT）编译器旨在确保CPU在任何交互发生之前都能理解代码中的每一行。
JIT本身不会使执行更快，因为它仍然执行相同的字节码序列。 但是，JIT允许在运行时进行优化。 一个好的JIT优化器会看到应用程序的哪些部分正在执行很多，称之为“热点”。 然后，它将通过用更高效的版本替换它们来优化那些代码。
这意味着当您的应用程序一次又一次地执行相同的操作时，它可以显着更快。 另外，请记住Java和C＃是强类型语言，因此优化器可以对代码做出更多假设。

PyPy有一个JIT，如前一节所述，它明显快于CPython。 这篇性能基准测试文章更详细 - []

**那么为什么Cpython不使用JIT？**
JIT存在缺点：其中一个是启动时间。 CPython启动时间已经相对较慢，PyPy比CPython启动慢2-3倍。 众所周知，Java虚拟机的启动速度很慢。 .NET CLR通过从系统启动开始来解决这个问题，但CLR的开发人员还开发了运行CLR的操作系统。
如果你有一个Python进程长时间运行，代码可以优化，因为它包含“热点”，那么JIT很有意义。
但是，CPython是一个通用的实现。 因此，如果您使用Python开发命令行应用程序，每次调用CLI时都必须等待JIT启动，这将非常缓慢。
CPython必须尝试尽可能多地使用用例。 有可能将JIT插入到CPython中，但这个项目基本停滞不前。
> 如果您想获得JIT的好处并且您有适合它的工作负载，请使用PyPy。

## “这是因为它是一种动态类型的语言”

在“静态类型”语言中，您必须在声明变量时指定变量的类型。 那些包括C，C ++，Java，C＃，Go。
在动态类型语言中，仍然存在类型的概念，但变量的类型是动态的。
``` python
a = 1
a = "foo"
```
在这个玩具示例中，Python创建了一个具有相同名称和`str`类型的第二个变量，并释放为第一个实例创建的内存。
静态类型的语言不是为了让你的生活变得困难而设计的，它们是按照CPU的运行方式设计的。 如果最终需要将所有内容都等同于简单的二进制操作，则必须将对象和类型转换为低级数据结构。
Python为你做到这一点，你永远不会看到底层执行，也不需要关心。

不必声明类型不是使Python变慢的原因，Python语言的设计使您几乎可以创建任何动态。 您可以在运行时替换对象上的方法，您可以将低级系统调用修补为在运行时声明的值。 几乎任何事都有可能。

正是这种设计使得优化Python非常困难。
为了说明我的观点，我将使用一个在Mac OS中运行的名为Dtrace的系统调用跟踪工具。 CPython发行版没有内置DTrace，因此您必须重新编译CPython。 我正在使用3.6.6进行演示
```sh
wget https://github.com/python/cpython/archive/v3.6.6.zip
unzip v3.6.6.zip
cd v3.6.6
./configure --with-dtrace
make
```
现在，Python.exe 将在代码中拥有Dtrace跟踪程序。[Paul Ross在Dtrace上写了一篇很棒的小演讲](https://github.com/paulross/dtrace-py#the-lightning-talk)。您可以下载python的[Dtrace启动文件](https://github.com/paulross/dtrace-py/tree/master/toolkit)来测量函数调用、执行时间、CPU时间、系统调用，以及各种各样的趣味程序。例如
```sh
sudo dtrace -s toolkit/<tracer>.d -c ‘../cpython/python.exe script.py’
```
`py_callflow`跟踪器显示应用程序中的所有函数调用

![Python callflow](https://cdn-images-1.medium.com/max/800/1*Lz4UdUi4EwknJ0IcpSJ52g.gif)

那么，Python的动态类型是会让它变慢吗？
* 比较和转换类型是昂贵的，每次读取变量，写入或引用类型时都要检查
* 很难优化一种如此动态的语言。 Python的许多替代品之速度如此之快的原因在于它们在性能追求而对灵活性做出了妥协
* 查看[Cython](http://cython.org/)，它结合了C-Static类型和Python来优化已知类型的代码，可以[提供84倍](http://notes-on-cython.readthedocs.io/en/latest/std_dev.html)的性能提升。

## 结论
> 其动态性和多功能性是导致Python主要缓慢的原因。 它可以用作各种问题的工具，可能有更多优化和更快的替代方案。
但是，有一些方法可以通过利用异步，理解分析工具以及考虑使用多个解释器来优化Python应用程序。
对于启动时间不重要且代码需要JIT的应用程序，请考虑PyPy。
对于性能至关重要并且有更多静态类型变量的代码部分，请考虑使用Cython。

## 进一步阅读
Jake VDP’s excellent article (although slightly dated) [https://jakevdp.github.io/blog/2014/05/09/why-python-is-slow/](https://jakevdp.github.io/blog/2014/05/09/why-python-is-slow/)

Dave Beazley’s talk on the GIL [http://www.dabeaz.com/python/GIL.pdf](http://www.dabeaz.com/python/GIL.pdf)

All about JIT compilers [https://hacks.mozilla.org/2017/02/a-crash-course-in-just-in-time-jit-compilers/](https://hacks.mozilla.org/2017/02/a-crash-course-in-just-in-time-jit-compilers/)


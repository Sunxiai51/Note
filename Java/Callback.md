# Callback

Callback，回调，名词，是回调函数、回调方法等的简称。

[Wikipedia](https://en.wikipedia.org/wiki/Callback_(computer_programming))中这样定义：
> In computer programming, a callback is any executable code that is passed as an argument to other code, which is expected to call back (execute) the argument at a given time. This execution may be immediate as in a synchronous callback, or it might happen at a later time as in an asynchronous callback.

译：
> 在计算机编程中, 回调是将在给定的时间执行的、作为参数传递给其他代码的任何可执行代码。此执行可能与同步回调中的即时相同, 也可能会在稍后的异步回调中发生。

本文中的回调是结合分层概念来说的，且只对Java中的回调进行阐述。

## 1. 分层与单向依赖原则

在软件设计与开发中，分层十分常见，例如：Web系统常用的三层结构——表现层、业务逻辑层和数据访问层；所编写的App使用了JDK或其他框架/类库；安装在操作系统上的应用程序调用操作系统的接口操作硬件，等等。上层使用下层的各种服务，而下层对上层一无所知，每一层都对自己的上层隐藏其下层的细节。

单向依赖原则：通常作为分层架构中的设计原则，除公共模块(不依赖其他模块的模块)可被各层模块调用外，一个层的模块只依赖于同一层或下层的模块，且依赖应该是单向的，不得循环或相互依赖。遵循该原则设计分层架构，可以明确各层职责，简化类关系。

## 2. Java中的回调函数

Java中的回调函数是上层模块实现的，将被下层模块(反过来)“执行”的函数，这个函数在上层模块中实现，但上层模块及更上层模块不会去调用。有如下两种应用场景：
- 上层模块向下层模块提供操作策略
- 上层模块从下层模块获得数据

### 2.1 上层模块向下层模块提供操作策略

举例：MBG运行时的日志输出

### 2.2 上层模块从下层模块获得数据

举例：进度数据更新

执行某个操作后，常常需要获取该操作的进度，而当这种操作在服务端进行时，客户端要知晓进度只能从服务端获取数据，通常获取数据的方式有两种：
- 轮询
- 通知
    - 好莱坞法则
---
title: KSCrash 源码阅读(1) Crash分类
date: 2021-12-29 20:03:55
tags:
---
## 1. 异常的类型

 *  Mach kernel exception：是指最底层的内核级异常。用户态?的开发者可以直接通过Mach API设置thread，task，host?的异常端口，来捕获Mach异常
 *  Fatal signal：又称 BSD? 信号，如果开发者没有捕获Mach异常，则会被host层的方法ux_exception()将异常转换为对应的UNIX信号，并通过方法threadsignal()将信号投递到出错线程。可以通过方法signal(x, SignalHandler)来捕获signal
 *  Uncaught C++ exception ： Captures and reports C++ exceptions
 *  Uncaught Objective-C NSException ：应用级异常，它是未被捕获的Objective-C异常，导致程序向自身发送了SIGABRT信号而崩溃，是app自己可控的，对于未捕获的Objective-C异常，是可以通过try catch来捕获的，或者通过NSSetUncaughtExceptionHandler()机制来捕获
 *  Deadlock on the main thread：
 *  User reported custom exception ：

 ## 2. Mach异常与 Unix 信号
 > - Mach是一个微内核，旨在提供基本的进程间通信功能。
 > - XNU是一个混合内核，由Mach微内核和更传统（“单片”）
 > - BSD unix内核的组件组成。它还包括在运行时加载内核扩展的功能（添加功能，设备驱动程序等）
 > - Darwin是一个Unix操作系统，由XNU内核以及各种开源实用程序，库等组成。
OS X是Darwin，加上许多专有组件，最着名的是它的图形界面API。
1. Mach微内核中有几个基础概念：

 - Tasks，拥有一组系统资源的对象，允许"thread"在其中执行。
 - Threads，执行的基本单位，拥有task的上下文，并共享其资源。
 - Ports，task之间通讯的一组受保护的消息队列；task可对任何port发送/接收数据。
 - Message，有类型的数据对象集合，只可以发送到port

2. signal：
 1) SIGHUP
本信号在用户终端连接(正常或非正常)结束时发出, 通常是在终端的控制进程结束时, 通知同一session内的各个作业, 这时它们与控制终端不再关联。
登录Linux时，系统会分配给登录用户一个终端(Session)。在这个终端运行的所有程序，包括前台进程组和后台进程组，一般都属于这个 Session。当用户退出Linux登录时，前台进程组和后台有对终端输出的进程将会收到SIGHUP信号。这个信号的默认操作为终止进程，因此前台进 程组和后台有终端输出的进程就会中止。不过可以捕获这个信号，比如wget能捕获SIGHUP信号，并忽略它，这样就算退出了Linux登录， wget也 能继续下载。
此外，对于与终端脱离关系的守护进程，这个信号用于通知它重新读取配置文件。
2) SIGINT
程序终止(interrupt)信号, 在用户键入INTR字符(通常是Ctrl-C)时发出，用于通知前台进程组终止进程。
3) SIGQUIT
和SIGINT类似, 但由QUIT字符(通常是Ctrl-)来控制. 进程在因收到SIGQUIT退出时会产生core文件, 在这个意义上类似于一个程序错误信号。
6) SIGABRT
调用abort函数生成的信号。
7) SIGBUS
非法地址, 包括内存地址对齐(alignment)出错。比如访问一个四个字长的整数, 但其地址不是4的倍数。它与SIGSEGV的区别在于后者是由于对合法存储地址的非法访问触发的(如访问不属于自己存储空间或只读存储空间)。
8) SIGFPE
在发生致命的算术运算错误时发出. 不仅包括浮点运算错误, 还包括溢出及除数为0等其它所有的算术的错误。
9) SIGKILL
用来立即结束程序的运行. 本信号不能被阻塞、处理和忽略。如果管理员发现某个进程终止不了，可尝试发送这个信号。
11) SIGSEGV
试图访问未分配给自己的内存, 或试图往没有写权限的内存地址写数据.
13) SIGPIPE
管道破裂。这个信号通常在进程间通信产生，比如采用FIFO(管道)通信的两个进程，读管道没打开或者意外终止就往管道写，写进程会收到SIGPIPE信号。此外用Socket通信的两个进程，写进程在写Socket的时候，读进程已经终止。
ios crash 中主要是SIGKILL,SIGSEGV,SIGABRT,SIGTRAP
引起系统信号crash 主要有内存泄露、野指针等.
几种特别类型:
0x8badf00d 在启动、终⽌止应⽤用或响应系统事件花费过⻓长时间,意为“ate bad food”。
0xdeadfa11 ⽤用户强制退出,意为“dead fall”。(系统⽆无响应时,⽤用户按电源开关和HOME)
0xbaaaaaad ⽤用户按住Home键和⾳音量键,获取当前内存状态,不代表崩溃
0xbad22222 VoIP应⽤用因为恢复得太频繁导致crash
0xc00010ff 因为太烫了被干掉,意为“cool off”
0xdead10cc 因为在后台时仍然占据系统资源(⽐比如通讯录)被干掉,意为“dead lock”

 ## 3. Objective-C Exception
比如我们经常遇到的数组越界，数组插入 nil，都是属于此种类型，主要包含以下几类：

- NSInvalidArgumentException
非法参数异常

- NSRangeException
数组越界异常

- NSGenericException
这个异常最容易出现在foreach操作中，在for in循环中如果修改所遍历的数组，无论你是add或remove，都会出错

- NSInternalInconsistencyException
不一致导致出现的异常
比如NSDictionary当做NSMutableDictionary来使用，从他们内部的机理来说，就会产生一些错误

- NSFileHandleOperationException
处理文件时的一些异常，最常见的还是存储空间不足的问题

- NSMallocException
这也是内存不足的问题，无法分配足够的内存空间





 

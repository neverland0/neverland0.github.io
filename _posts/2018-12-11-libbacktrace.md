---
layout: item
title: 使用libbacktrace获取堆栈信息
---

## 堆栈信息有什么用

  程序被执行时,程序会被加载到操作系统为进程构造的虚拟地址中，虚拟地址的低地址处存放的是实现函数调用时使用的栈，发生函数调用时，传给被调用函数的参数，以及从被调用函数返回的地址等信息被压入栈中，有两个指向栈顶和栈底的指针分别被保存在寄存器中，每个函数对应一段栈帧，其中指向当前栈帧底部的称为*帧指针* ，指向栈帧增长方向顶部的称为*栈指针* ，这样在被调用函数被执行时，通过相对于帧指针的偏移量，就能获得调用函数传递给它的参数，以及函数执行完之后的返回地址。

  ![zhongshan](/img/stack.gif)

  同时我们注意到，帧指针指向的位置还保存了一个地址信息，而这个信息就是上一个栈帧对应的帧指针的地址，这样通过当前的帧指针，我们还能得到上一个栈帧的帧指针的地址，而这也是栈展开能够实现的原因，所以简单的说，堆栈中保存了函数调用链的信息，而函数调用关系可以帮助我们了解一个程序的执行原理或是调试程序。

## “静态”获取堆栈信息

  当我们在UNIX环境下写程序时，如果我们的程序展现出一些非法行为时，操作系统内核出于保护目的，会向进程发送终止信号，对于有些行为，会把进程当前的使用的虚拟内存信息保存到一个文件中，以供我们分析，这就是"core dump"，我们会得到一个core文件，如果我们想知道我们的程序运行到哪里奔溃了，那么我们通常会使用gdb工具调试core文件，进入gdb交互界面后，输入"where"，gdb就能将程序崩溃时，也就是core文件中对应的堆栈信息打印出来，每一帧对应于一个函数调用，这样我们就可以分析出程序崩溃的原因。

## 理发师如何给自己理发

  但调试core文件的方法存在一些问题，比如由于系统设置的问题或是core文件太大，从而导致无法生成core文件，那么我们能不能让程序崩溃时自己把堆栈信息打印出来，毕竟大家都是喜欢自动化的东西。这听上来有点像要求一个理发师自己给自己理发一样，也不是不可能，但就是比较困难，毕竟堆栈的设计是为了实现程序中的函数调用，而我们现在又要在程序中获取这一底层信息。如果从编译器的层面来考虑应该是比较简单的，但编译器的主要目的还是给程序员提供了一种抽象，所以尽管可以做，但实际上并没有特定的方法，而gcc也确实提供了一个函数backtrace()，这个函数也能实现一部分功能，为什么说是一部分，因为对于一些涉及到动态库中函数信息的情况，backtrace()就无能为力了。看来gcc不想让我们太关注底层的一些细节，所以我们的“理发师”来了，gcc版go编译器前端的作者[Ian Lance Taylor](https://www.airs.com/ian/)实现了库[libbacktrace](https://github.com/ianlancetaylor/libbacktrace),这个库提供和gcc标准库中backtrace()类似的功能，但却更好用，也更灵活，下面就简单介绍一下这个库的一些接口和使用。

## libbacktrace

  下载后执行
- ./configure && make && make install

  完成安装。
  下面是主要接口说明，详细说明可以参考源文件中的backtrace.h.

```c++

//保存backtrace状态的结构体
struct backtrace_state;

//创建backtrace状态，必须最先被调用，而且返回值要传给backtrace_full()等例程。
//第一个参数要指定当前可执行文件名。
struct backtrace_state *backtrace_create_state (const char *filename, 
                                                int threaded,
                                                backtrace_error_callback,
                                                error_callback, 
                                                void *data);

//主要使用的函数，调用该函数，传入之前得到的backtrace状态结构体，对于栈帧中每一帧都会调用回调函数一次。
//在回调函数中会得到每一个栈帧需要的信息。
int backtrace_full (struct backtrace_state *state, 
                    int skip,
                    backtrace_full_callback callback,
                    backtrace_error_callback error_callback,
                    void *data);

//backtrace_full例程的回调函数，参数filename、lineno、function分别是当前栈帧中函数的文件名、行号、函数名。
typedef int (*backtrace_full_callback) (void *data, 
                                        uintptr_t pc,
                                        const char *filename, 
                                        int lineno,
                                        const char *function);

//backtrace_full例程的错误回调函数，当backtrace_full函数发生错误时，该函数被调用，可以不使用。
typedef void (*backtrace_error_callback) (void *data, 
                                          const char *msg,
                                          int errnum);
```

  使用方法：
  
1. 先调用backtrace_create_state()例程，参数filename是可执行文件名，参数threaded非零表示支持多线程，否则只支持单线程。
2. 将得到的返回值保存下来，在需要打印堆栈信息的地方调用backtrace_full()例程，第一个参数为之前得到的返回值，第三个参数为需要自己实现的回调函数，对于每一个栈帧，回调函数都会被调用一次。
3. 在自己实现的回调函数中，参数filename、lineno、function分别为函数对应的文件名、行号、函数名，然后按照自己的需求处理即可。


## 例子
  如果这些枯燥的文字让你感到乏味，你可以看一下这个用libbacktrace实现的一个单例模式的栗子[StackPrinter](https://github.com/neverland0/StackPrinter)，这并不是一个可以直接编译运行的源代码，你需要首先安装libbacktrace并做一些适当的修改，但这些并不关键。


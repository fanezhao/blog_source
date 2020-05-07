---
title: Go并发编程（五）IPC之信号
tags:
  - Go
date: 2020-04-22 00:18:30
---


> 我在[多进程编程](http://zmoyi.com/2020/04/17/GoConcurrent2-MultiProgress/)那篇博客中讲到Go支持的IPC方法有管道、信号和socket。今天来看下在Go中是如何使用信号的。

## 什么是信号

操作系统信号（signal）是IPC方法中唯一一种异步的通信方法，它的本质是用软件来模拟硬件的中断机制。

信号是用来通知某个进程有某个事情发生了。例如，在命令行终端按下某些快捷键，就会挂起或停止正在运行的程序；或者通过`kill`命令杀死某个进程的操作也有信号参与。

每个信号都是一个以“SIG”为前缀的名字，例如`SIGINT`、`SIGQUIT`以及`SIGKILL`等。但其实，在操作系统内部，信号是由正整数表示的，这些正整数称为**信号编号**。在Linux在可以通过`kill`命令查看当前系统所支持的信号：

```shell
$ kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX
```

可以看到，Linux支持的信号有62种（编号32和33的信号为空）。

### 标准信号和实时信号

- 标准信号：编号1到32的信号属于标准信号（也叫不可靠信号）。
- 实时信号：编号34到64的信号属于实时信号（也叫可靠信号）。

其中标准信号存在两个问题：

- 1、对于同一个进程来说，每种标准信号只会被记录并处理一次。
- 2、如果发送给某一个进程的标准信号的种类有多个，那么它们的处理顺序是不确定的。

实时信号解决了标准信号上述的两个问题，即多个同种类的实时信号都可以记录在案，并且可以按照信号的发送顺序被处理。虽然实时信号在功能上更加强大，但是已经成为事实标准的标准信号也没法被替换，所以两者一直共存着了。

简单来说，信号的来源有键盘输入（比如Ctrl+c）、硬件故障、系统函数调用和软件中的非法运算。进程响应信号的方式有三种：**忽略、捕捉和执行默认操作。**

Linux对每一个标准信号都默认的操作方式。针对不同各类的标准信号其默认的操作方式一定是下列操作之一：**终止进程、忽略该信号、终止进程并保存内存信息、停止进程、恢复进程（若进程已停止）。**

## 使用Go中开发处理信号程序

### 自定义信号处理

对于绝大多数标准信号而言，我们可以自定义程序对它的响应方式。更具体的讲，进程要告知操作系统内核：当某种信号到来时，需要执行某种操作。在程序中，这些自定义的信号响应方式往往由函数表示。

Go命令会对其中的一些以键盘输入为来源的标准信号做出响应，这是通过`os/signal`中的一些API来实现的。下面就从接口类型`os.Signal`开始讲起：

```go
type Signal interface {
  String() string
  Signal() // to distinguish from other Stringers
}
```

`其中Signal`方法的声明没有实际意义，它只是作为`os.Signal`接口的一个标识。在Go的标准库中，所有实现它的类型的`Signal`方法都是空方法。所有实现此接口类型的值都可以表示成一个操作系统信号。

另外，查看`syscall.Signal`类型的`String`方法的源码，会发现在其中有一个数组类型的私有变量`signals`。它的每个索引值都代表了一个标准信号的编号，而对应的元素则是针对该信号的一个简短描述。

代码包`os/signal`中的`Notify`函数用来当操作系统向当前进程发送指定信号时发出通知。

```go
func Notify(c chan<- os.Signal, sig ...os.Signal)
```

- 第一个参数是通道类型的。在`Notify`函数中，只能向它发送`os.Signal`类型的值（以下简称信号值），而不能从中获取信号值。`Notify`函数会把当前进程接收到的信号放入这个通道（以下简称`signal`接收通道）中，这样该函数的调用方就可以从这个`signal`接收通道中按顺序获取操作系统发来的信号并进程相应的处理了。

- 第二个参数是可变长的参数，它代表的参数值包含我们希望自行处理的所有信号。接收到需要自行处理的信号之后，`os.Signal`中程序会把它封装成`os.Signal`类型的值并放入到`signal`接收通道中。

我们可以只为第一个参数绑定实际值，在这种情况下，`signal`处理程序会理解为我们想要自行处理所有信号，并把接收到的几乎所有信号全部封装到`signal`接收通道中。举个粟子：

```go
sigRecv := make(chan os.Signal, 1)
sigs := []os.Signal{syscall.SIGINT, syscall.SIGQUIT}
signal.Notify(sigRecv, sigs...)
// 永远都不会停止
for sig := range sigRecv {
  // 接收到信号不做任何处理，直接输出，相当于忽略信号，因此永远不会停止，除非kill掉
  fmt.Printf("%s\n", sig)
}
```

在这个程序中，我们希望自行处理`SIGINT`和`SIGQUIT`信号。所以，`sigs`包含了`syscall.SIGINT`和`syscall.SIGQUIT`。然后，从`signal`接收通道中接收信号值，并做打印处理。

在实际场景中，上述的做法是比较危险的，因为我们忽略了当前进程本该处理的信号。如果当前进程接收到了未自定义处理方法的信号，就会执行由操作系统指定的默认操作。因此如果指定的自定义处理方法只是打印一些内容，就相当于使当前进程忽略掉相应的信号。这也就是如果我们使用`Ctrl+c`无法停掉上面的程序而只会在控制台打印的原因。

如果上述代码改成这样：

```go
sigRecv := make(chan os.Signal, 1)
signal.Notify(sigRecv)
for sig := range sigRecv {
  fmt.Printf("%s\n", sig)
}
```

这段程序表示该进程会忽略掉所有信号，这样导致的后果会很悲剧，我们也无法通过`Ctrl+c`停止程序。

不过，还好在类Unix操作系统下在`SIGKILL`和`SIGSTOP`两种信号既不能自行处理，也不会被忽略，对它们的响应只能是执行系统的默认操作。这种策略的根本原因是：**它们向系统的超级用户提供了使进程终止或停止的可靠方法。**即使我们在程序中调用：

```go
signal.Notify(sigRecv, syscall.SIGKILL, syscall.SIGSTOP)
```

也不会改变当前进程对`SIGKILL`和`SIGSTOP`两种信号的处理动作。这也是我们无法通过`Ctrl+c`而可以通过`kill`命令停止上述程序的原因。

### 恢复系统默认信号处理

对于除`SIGKILL`和`SIGSTOP`两种外，除了能够自行处理它们之外，还可以在之后的任意时刻恢复对它们的系统默认操作。这需要用到`os/signal`包中的`Stop`函数：

```go
func Stop(c chan <- os.Signal)
```

其中只有一个参数声明，并且和`signal.Notify`函数的第一个参数声明完全一致。这并不是巧合，而是有意为之。

只有把当初传递给`signal.Notify`函数的那个`signal`接收通道作为调用`signal.Stop`函数的参数值，才能如愿以偿的取消掉之前的行为，否则调用`signal.Stop`不会起到任何作用。

调用完`signal.Stop`函数之后，作为其参数的`signal`接收通道将不会再被发送任何信号。这时，会在之前第一段代码的从`signal`接收通道中接收信号值的for语句会一起被阻塞。不过我们可以在调用`signal.Stop`之后使用内建函数`close`关闭它：

```go
signal.Stop(sigRecv)
close(sigRecv)
```

在很多时候，我们并不想完全取消掉自行信号处理信号的行为，而只是想取消一部分。为此，我们只需要再次调用`signal.Notify`函数，并重新设定其第二个可变参数值即可。不过要保证作为第一个参数的`signal`接收通道相同。

**问题：如果`signal`接收通道不同，会怎样？**

如果我们先后调用了两次`signal.Notify`函数，但两次传递给该函数的`signal`接收通道不同，那么`signal`处理程序会将这两次调用视为毫不相干。

## 总结

本篇文章我们学习信号的一些基础概念，还有在Go中如何对信号进行自定义处理，以及如何恢复部分信号为系统默认处理等知识。

## 链接

本篇文章中的但到参考代码并不算多，在我的Github上有一个[完整示例](https://github.com/fanezhao/GoConcurrent/tree/master/chapter3/signal)，大家可以点击查看运行。


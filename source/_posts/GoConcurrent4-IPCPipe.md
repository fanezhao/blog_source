---
title: Go并发编程（四）IPC之管道
date: 2020-04-19 01:27:00
tags:
 - Go
---

> 我在[多进程编程](http://zmoyi.com/2020/04/17/GoConcurrent2-MultiProgress/)那篇文章中讲到Go支持的IPC方法有管道、信号和socket。今天来看下在Go是如何支持管道的。

## 什么是管道

管道是一种半双工（或者是单向）的通信方式，**只能用于父进程与子进程或同祖先的子进程之间的通信。**例如，在使用Shell命令的时候，常常就会用到管道，例如：

```shell
$ ps aux | grep go
```

shell为每个命令都创建一个进程，然后把左边命令的标准输出用管道与右边命令的标准输入连接起来。管道的优点在于简单，而缺点则是只能单向通信以及通信双方关系上的严格限制。

## 在Go中如何使用



### 简单命令

对于管道，Go中是通过`os/exec`中的API支持的，我们可以执行操作系统命令并在之上建立管道。下面创建一个exec.Cmd类型的值：

```go
cmd0 := exec.Command("echo", "-n", "My first command comes from golang.")
```

对应的shell命令就是：

```shell
$ echo -n "My first command comes from golang."
```

在exec.Cmd类型之上有个Start方法，可以用它启动命令：

```go
if err := cmd0.Start(); err != nil {
  fmt.Printf("error %s\n", err)
  return
}
```

此外，还要创建一个能够获取上述命令的输出管道：

```go
stdout0, err := cmd0.StdoutPipe()
if err != nil {
  fmt.Printf("error %v\n", err)
  return
}
```

变量`cmd0`的`StdoutPipe`方法会返回一个输出管道并赋值给`stdout0`，`stdout0`的类型是`io.ReadCloser`，它是一个扩展了`io.Reader`接口的接口类型，并定义了可关闭的数据读取行为。

有了`stdout0`，启动上述命令之后，就可以通过调用它的`Read`方法来获取命令的输出了：

```go
output0 := make([]byte, 30)
n, err := stdout0.Read(output0)
if err != nil {
  return
}
fmt.Printf("%s\n", output0[:n])
```

这里的`Read`方法会把读出的输出数据存入字节切片`output0`中，并返回一个`int`和`error`类型的值。如果命令的输出小于`output0`的长度，那么变量n的值会是命令实际输出的字节数量，否则n的值就等于`output0`的长度。

**而后一种情况则说明输出管道中的数据没有被完全读出，这时就需要再次读取一次或多次，直至读完为止。**在读完的时候第二个返回值会是`io.EOF`，我们可以以此判断读取情况，这里把切片的长度设置的很小，以便达到迭代读取的效果：

```go
var outputBuf0 bytes.Buffer
for {
  tempOutput := make([]byte, 5)
  n, err := output0.Read(tempOutput)
  if err != nil {
    if err == io.EOF {
      break
    } else {
      return
    }
  }
  if n > 0 {
    outputBuf0.Write(tempOutput[:n])
  }
}
fmt.Printf("%s\n", outputBuf0.String())
```

缓冲区`outputBuf0`用来收集每次迭代读取出来的内容。

还有一个更加便捷的方式，就是使用**缓冲读取器**从管道中读取数据，像下面这样：

```go
outputBuf0 := bufio.NewReader(output0)
output0, isPrefix, err := outputBuf0.ReadLine()
if err != nil {
  return
}
fmt.Printf("%v, %s\n", isPrefix, string(output0))
```

由于`output0`也是`io.Reader`类型的，所以可以直接把它作为`bufio.NewReader`的参数。这个函数会返回一个`bufio.Reader`类型的值，也就是一个缓冲读取器。默认情况下，该读取器会带一个长度为4096的缓冲区，而这个缓冲区的长度就代表一次可以读取的字节的最大数量。由于`cmd0`的命令只会输出一行内容，所以可以直接用`outputBuf0`的`ReadLine`方法读取。

这个方法的第二个bool类型的值代表当前行是否还未读完，如果它是false，可以像前面一样利用for循环继续读取剩余数据。不过在这里并不需要。

使用缓冲读取器的好处就是可以非常灵活方便的读取需要的内容，而不是只能先把所有内容都读出来再处理。

### 复杂命令

除了上面讲的实现一些简单命令外，管道还可以实现一些复杂的命令。比如，把一个命令的输出作为另外一个命令的输入，用Go的话实现起来非常简洁。例如，下面有两个exec.Cmd类型的值：

```go
cmd1 := exec.Command("ps", "aux")
cmd2 := exec.Command("grep", "apipe")
```

首先，设置`cmd1`的`Stdout`字段，然后启动`cmd1`，并等待它运行完毕：

```go
var outputBuf1 bytes.Buffer
cmd1.Stdout = &outputBuf1
if err := cmd1.Start(); err != nil {
  return
}
if err := cmd1.Wait(); err != nil {
  return
}
```

这里使用了字节序列缓冲区`&outputBuf1`来接收`cmd1`的输出，是因为`*bytes.Buffer`类型实现了`io.Writer`接口。这样，在`cmd1`命令启动的时候，所有的输出内容都可以写到`outputBuf1`中了。

另外，`cmd1`的`Wait`方法会一直阻塞，直到`cmd1`命令运行结束。

接下来，再设置`cmd2`的`Stdin` 和`Stdout`字段，启动`cmd2`并等待它运行完毕：

```go
cmd2.Stdin = &outputBuf1
var outputBuf2 bytes.Buffer
cmd2.Stdout = &outputBuf2
if err := cmd2.Start(); err != nil {
  return
}
if err := cmd2.Wait(); err != nil {
  return
}

fmt.Printf("%s\n", outputBuf2.String())
```

同样，这里由于`*bytes.Buffer`类型也实现了`io.Reader`接口，我才能把`&outputBuf1`也赋值给`cmd2`的`Stdin`字段。最后，为了获得`cmd2`的输出内容，需要等它运行完毕之后再到`outputBuf2`中查看。

综上，因为我们对`&outputBuf1`再次赋值，将`cmd1`的输出和`cmd2`的输入串联到了一起，模拟出了操作系统的命令：

```shell
$ ps -aux | grep apipe
```

需要注意的是`cmd2`的输出和直接在操作系统上运行这个命令的的输出会有所不同。是因为该程序在自身运行的时候又运行了上面的这个命令。

## 匿名管道和命名管道

刚刚，上面所讲到的管道也可以叫做**匿名管道**，与此相对应的是**命名管道（named pipe）**。

和匿名管道的区别在于，**任何进程都可以通过命令管道交换数据**。而匿名管道**只能用于父进程与子进程或同祖先的子进程之间的通信**。实际上，命名管道是以文件的形式存在于文件系统中，使用它的方法与使用文件很相似。

需要注意的是，命名管道默认是阻塞式的。具体的说就是，只有在对这个命名管道的读操作和写操作都准备就绪之后，数据才开始流转。还要注意，命名管道仍然是单向的，又由于可以实现多路复用，所以有时候也需要考虑多个进程同时向命名管道写数据的情况下的操作原子性问题。

在Go的标准库代码包`os`包中，包含了可以创建这种独立管道的API。例如：

```go
reader, writer, err := os.Pipe()
```

函数`os.Pipe`会返回三个返回值。

- `reader`代表了该管道的输出端的`*os.File`类型的值。
- `writer`代表了该管道的输出端的`*os.File`类型的值。
- `err`代表可能发生的错误。若没有，则其为nil。

Go使用系统函数来创建管道，并把它的两端封装成两个`os.File`类型的值。例如这样两段代码：

```go
n, err := writer.Write(input)
if err != nil {
  fmt.Printf("Error: Couldn't write data to the named pipe: %s\n", err)
}
fmt.Printf("Written %d byte(s). [file-based pipe]\n", n)
```

和

```go
output := make([]byte, 100)
n, err := reader.Read(output)
if err != nil {
fmt.Printf("Error: Couldn't read data from the named pipe: %s\n", err)
}
fmt.Printf("Read %d byte(s). [file-based pipe]\n", n)
fmt.Printf("%s\n", string(output))
```

如果它们是并发运行的，那么在`reader`之上调用`Read`方法就可以按顺序获取得到之前通过调用`writer`和`Write`方法写入的数据。**为什么强调是并发运行？**因为命名管道默认会在其中一端还未准备就绪的时候阻塞另一端。所以如果顺序执行这两段代码，那么程序肯定会被永远阻塞在：

```go
n, err := writer.Write(input)
```

或

```go
n, err := reader.Read(output)
```

出现的地方。具体阻塞在哪取决于两个表达式`writer.Write(input)`和`reader.Read(output)`哪一个先被求值。

另外，因为管道都是单向的，所以虽然`reader`和`writer`都是`*os.File`类型的，但是却不能调用`reader`的那些写方法或`writer`的那些读方法，否则就会得到非`nil`的错误值，会被告知这样是不允许的。

由于通过`os.Pipe`函数生成的管道在底层是由系统级别的管道来支持，所以使用的时候要注意操作系统对管道的限制。例如，匿名管道会在管道缓冲区写满之后阻塞写进程。以及命名管道会在其中一端未就绪的情况下阻塞另一端的进程，等等。

再次强调，命名管道可以被多路复用。所以，当有多个输入端同时写入数据时，就不得不考虑操作原子性的问题。

## 原子性操作的管道

前面我们说操作系统提供的管道是不支持原子操作的。为此Go在`io`包中提供了一个基于内存的有原子操作保证的管道，简称**内存管道**。

```go
reader, writer := io.Pipe()
```

函数`io.Pipe`有两个返回值：

- `reader`代表了该管道输出端的`*io.PipeReader`类型的值。
- `writer`代表了该管道输入端的`*io.PipeWriter`类型的值。

这两个类型的值分别对管道的输入和输出端做了很好的限制，可以避免我们对管道的反向使用。还有，在使用`Close`方法关闭管道的某一端之后，另一端会得到 一个预定义的`error`类型的错误值。我们还可以通过调用`CloseWithError`来自定义这种错误。

另外，还需要注意的是，跟`os.Pipe`函数生成的管道一样的是，我们仍需要并发执行分别对内存管道两端进行操作的那两块代码。

**内存管道的高效**

在内存管道内部，充分利用了`sync`代码包中的API，并以此从根本上保证了相关操作的原子性，所以我们可以放心的并发的使用它。另外，由于这种管道不是基于文件系统的，并没有作为中介的缓冲区，所以通过它传递的数据只会被复制一次，这也就更进一步提高了数据传输效率。

## 总结

本篇文章介绍系统级别的匿名管道和命名管道的概念和基本用法，以及Go标准库中与它们对应的API。另外，还简单说明了Go特别提供的一种基于内存的同步管道的使用方法。

## 链接

[文章代码](https://github.com/fanezhao/GoConcurrent/tree/master/chapter3)。




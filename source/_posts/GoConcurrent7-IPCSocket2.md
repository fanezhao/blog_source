---
title: Go并发编程（七）IPC之socket举个粟子
tags:
  - Go
date: 2020-04-24 18:00:00
---


> 在上一篇文章[IPC之socket基础](http://zmoyi.com/2020/04/24/GoConcurrent6-IPCSocket1/)中，我们学习了socket的基础概念，也一并学习了在Go中相关API。这篇文章，我们主要通过详细案例来加深对上篇文章理解。
>
> 之所以没有在上一篇文章中说这个粟子，主要是害怕内容太长，容易引起不适（太长不看系列）。

## 粟子功能

该示例包含了服务端和客户端程序，它们以网络和TCP协议作为通信的基础。服务端程序功能可以概括为：接收客户端程序的请求，计算请求数据的立方根，并把结果返回给客户端程序。细节如下：

- 需要根据事先约定好的数据边界把接收到的数据切分成数据块。
- 仅可接受类型为`int32`表示的请求数据块。对于不符合要求的数据块，要生成错误信息并返回给客户端程序。
- 对于每个符合要求的数据块，需要计算它们的立方根。生成结果描述符并返给客户端程序。
- 需要鉴别闲置的通信连接并主动关闭它们。闲置的连接紧鉴别依据：在过去的10s内，没有任何数据经该连接传送到服务端。这可以在一定程度是节约相关资源。

客户端的功能相对简单一些，可以概括为：向服务端发送若干实为整数的数据请求，接收服务器返回的响应并记录他们。细节如下：

- 发送给服务端的每块请求数据都带有约定好的数据边界。
- 需要根据事先约定好的数据边界把接收到的响应切分成数据块。
- 在获得所有期望的响应数据之后，应该及时关闭连接以节省资源。
- 需要严格限制耗时，从开始向服务端程序发送数据到接收到所有期望的响应数据，其耗时不应该过过眼超过5s，否则在报告超时错误之后关闭连接。这实际上是对服务端程序的响应速度的检验。

## 粟子代码

这个粟子的代码不少，干脆就不在这里展示了，放在了我的Github上，链接在最下面，有兴趣可以点击查看。

## 总结

总结还是要总结一下的，这章主要就是通过举例来加深对`socket`的理解，以及在Go中与`socket`相关API的使用。

在Go标准库中，一些实现了某种网络通信功能的代码包都是以`net`代码包所提供的`socket`编程API为基础的。其中最有代表性的就是`net/http`代码包，它以此为基础实现了`TCP/IP`协议栈的应用层协议`HTTP`，并提供了非常好用的API。它可以满足大多数应用程序的编写。

除了`net`代码包之外，标准库中`net/rpc`包的API为我们提供了在两个Go程序之间建立通信和交换数据的另一种方式，这种方式也称**远程过程调用（Remote Procedure Call）**。这个代码包中的程序也是基于`TCP/IP`协议栈的，它们也用到了`net`包以及`net/http`包提供的API。

## 链接

[举个粟子](https://github.com/fanezhao/GoConcurrent/blob/master/chapter3/socket/tcp_socket.go)，注释超丰富。

---
title: 超高性能管线式HTTP请求
date: 2018-06-23 18:04:35
categories: 架构方案
tags:
    - 性能效率
---

这里的高性能指的就是网卡有多快请求发送就能有多快，基本上一般的服务器在一台客户端的压力下就会出现明显延时。

该篇实际是介绍pipe管线的原理，下面主要通过其高性能的测试实践，解析背后数据流量及原理。最后附带一个简单的实现

**实践**

先直接看对比测试方法。

测试内容单一客户的使用尽可能快的方式向服务器发送一定量（10000条）请求，并接收返回数据。

对于单一客户端对服务器进行http请求，一般我们的方式：

1：单进程或线程轮询请求（这个效能自然很低，原因会讲到，也不用测试）

2：多条线程提前准备数据等待信号（对客户端性能要求较高）

3：提前准备一组线程同时轮询操作

4：使用系统/平台自带异步发送机制（实际就是平台线程池的方式，发送与接收使用从线程池中的不同线程）

 

对于测试方案1，及方案2测试中性能较低没有可比性，后面测试不会展示其结果。

以下展示后面2种测试方法及当前要说的管线式的方式：

- 先讲管线式（pipe）测试方案（原理在后面会讲到），测试中使用100条管线（管道），实际上更少甚至一条管线也是能达到近似的性能，不过多数服务器nginx限制一条管可以持续发送request的数量（大部分是100也有部分会是200或是更高），每条管线发送100个请求。
- 然后是线程组的方式准备100条线程（100条线程并不是很多不会对系统本身有明显影响），每条线程轮询发送100个request。
- 异步方式的方式，10000全部提交发送线程，由线程池控制接收。

 

测试环境：普通家用PC，i5 4核，12G ，100Mb电信带宽

 

测试数据：

GET http://www.baidu.com HTTP/1.1

Content-Type: application/x-www-form-urlencoded

Host: www.baidu.com

Connection: Keep-Alive

 

这里就是测试最常用的baidu，如果测试接口性能不佳，大部分请求会在应用服务器排队，难以直观提现pipe的优势（其实就是还没有用到pipe的能力，服务器就先阻塞了） 

 

下文中所有关于pipe的测试都是使用PipeHttpRuner  （http://www.cnblogs.com/lulianqi/p/8167843.html为该测试工具的下载地址，使用方法及介绍）

 

先直接看管道式的表现：（截图全部为windows自带任务管理器及资源管理器）

![](https://mmbiz.qpic.cn/mmbiz_png/icNyEYk3VqGkiapBYduvX3Xq2BbVktQQNvvzfxwbwaibAK7ibXU7dkXqehoD9EGRZtQvQcpTrzbJQoawRibUn5srNsg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

![](https://mmbiz.qpic.cn/mmbiz_png/icNyEYk3VqGkiapBYduvX3Xq2BbVktQQNvwd9xymTmapyGpfL5EFNl7FicvXygviaIKkyiaicibH4YcI68D6tGPKxicwYw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

先解释下截图含义，后面的截图也都是同样的含义：

 

第一副为任务管理器的截图实线为接收数据，虚线为发送数据,取样0.5s，每一个正方形的刻度为1.5s(因为任务管理器绘图策略速率上升太快过高的没有办法显示，不过还是可以看到时间线）；

第二副为资源管理器，添加了3个采样器，红色为CPU占用率，蓝色为网络接收速率，绿色为网络发送速率。

 

测试中 一次原始请求大概130字节，加上tcp，ip包头，10000条大概也只有1.5Mb（包头不会太多因为管道式请求里会有多个请求放到一个包里的情况，不过大部分服务器无法有这么快的响应速度会有大量重传的情况，实际上传流量可能远大于理论值）。

一次的回包大概在60Mb左右（因为会有部分连接中途中断所以不一定每次测试都会有10000个完整回复）。

 

可以看到使用pipe形式性能表现非常突出，总体完成测试仅仅使用了5s左右。

发送本身压力比较小，可以看到0.5秒即到达峰值,其实这个时候基本10000条request已经发送出去了，后面的流量主要来自于服务器端缓存等待（TCP window Full）来不及处理而照成是重传，后面会讲到。

再来看看response的接收，基本上也仅仅使用了0.5s即达到了接收峰值，使用大概5s 即完成了全部接收，因为测试中cpu占用上升并不明显，而对于response的接收基本上是从tcp缓存区读出后直接就存在了内容里，也没有涉及磁盘操作（所以基本上可以说对于pipe这个测试并没有发挥出其全部性能，瓶颈主要在网络带宽上）。

 

再来看下线程组的方式（100条线程每条100次）：

![](https://mmbiz.qpic.cn/mmbiz_png/icNyEYk3VqGkiapBYduvX3Xq2BbVktQQNvseAUS4vIEDgGMhfCt6FEPPWQ39fKqTKD4ub379W1Yp4ZYXUB0TDmzg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

![](https://mmbiz.qpic.cn/mmbiz_png/icNyEYk3VqGkiapBYduvX3Xq2BbVktQQNvFQibribGiaB0OwjOKhDiblVXOQkBjzMMJ55dPwPjZuFzOiatb5FLAS7S8WQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)


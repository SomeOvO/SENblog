---
title: Go RTMP服务器研究
published: 2026-08-16
description: RTMP库（Joy4 ）的一个使用示例
image: ""
tags:
  - 开发日记
category: 我思
draft: false
lang: zh-CN
---
# 介绍

在很久前我设想过一个系统。

然而近期闲来无事，想起了这个搁置的想法，不过目前也没有这个需求了。只聊一下可行性吧，不然博客真就没内容了。

在游玩一些竞技游戏的时候，难免需要一些复盘的环节，当然你如果到了这个环节多半红温了。

这个时候就需要一个系统来进行复盘，在常规的复盘过程中，所有人都几乎各执一词，这就造成需要一个系统，**将所有人的回放**整合起来。

当时构想的思路是这样的：

```
客户端（OBS推流工具）
↓
服务器（保存处理回放）
↓
客户端（查看回放）
```


显然客户端的处理不需要我们来进行，所以只需要写服务端即可。

在服务端，你要做的事情无非以下几种：

1. 接收来自客户端推流
2. 处理推流数据，将其保存至本地或者通过HTTP实时访问
3. 在用户需要流媒体文件的时候进行处理，发送。

这里的推流协议，我选用了常规视频推流的协议：

# RTMP

[维基百科](https://en.wikipedia.org/wiki/Real-Time_Messaging_Protocol)

在最初实现中，我构想出采用[SRS](https://github.com/ossrs/srs)作为视频接收服务器，然后写一个程序持续获取流。

然而，问题是怎么获取流呢？以及单独部署一个SRS服务端过于臃肿。

在最初获取流的时候，我采用ffmpeg来下载流文件，但是当时的问题在于ffmpeg的启动方式是通过命令行，且无法有效关闭ffmpeg进程。现在看来当时实在是想太多了......

我把目光放在纯Go语言实现，因为我只会这个；话说是不是要学一些C了？

我所使用的库为[joy4 | Golang audio/video library and streaming server](https://github.com/nareix/joy4)

不过这个库似乎更新到[joy5](https://github.com/nareix/joy5)了，但是由于joy5缺少示例，我还是使用joy4作为演示：

首先，你要启动一个RTMP服务吧！


# 启动RTMP服务

```
package main

import (
    "github.com/nareix/joy4/format/rtmp"
)

  

func main(){
    server := &rtmp.Server{}

    server.ListenAndServe()
}
```

恭喜你，成功启动了一个RTMP服务器。

不过这时候你可能会好奇：是不是少了点什么？

当然，你如果想自定义端口的话请在`rtmp.Server{}`中加入`Addr`字段：`":7891"`

好了不开玩笑了，这只是启动了一个RTMP服务器，服务器的处理方法还没有写。

给`server.HandlePublish`字段赋值一个函数


```
func main() {

    server := &rtmp.Server{}

    server.HandlePublish = OnPublish

    server.ListenAndServe()

}

  

func OnPublish(c *rtmp.Conn) {

  

}
```

这个函数是代表服务端接收到推流后处理的事件

那么有人就会问了，我要怎么处理这些事件呢？


把处理过程分为两部分：读，写

## ？！读读？！

读取什么呢？

### 路径 URL.Path

首先，先读取推流路径，当然其他的也可以，来创建一个属于此次推流的键值（Key）

`Path := c.URL.Path`

这里的`c.URL`返回的是一个 **url.URL** 指针，指向推流地址；在示例中，通过路径来当作键值。

然后再读取什么呢？

### 元数据  CodecData

通过 `Streams, err := c.Streams()` 获取推流的编码器数据

## ？！写写？！

写什么呢？一个很现实的问题放在我们脸上......写到哪里呢？

在示例中，作者给出了一个流程：

接收推流 -> 创建Channel -> 向Channel持续写入数据 <- 其他服务读取数据即可

这就解释了为什么要指定一个键值。

那么我们如何让程序写入呢？

创建一个切片

```go
type Channel struct {

    Que *pubsub.Queue

}


var Channels = map[string]*Channel{}
```

读取切片内容，如果切片不为nil，则上一次推流未被销毁，进行销毁操作然后重新分配

```go
    ch := Channels[Path]

    if ch != nil {

        ch = nil

    }

    ch = &Channel{}

    ch.Que = pubsub.NewQueue()

    err = ch.Que.WriteHeader(Streams)

    if err != nil {

        LE("向频道写入数据流出错", err)

    }

    Channels[Path] = ch
```

可以看到，目前只是创建了一个频道，然后初始化了一些元数据，并没有实际进行数据操作。

下一步我们将使用 `avutil.CopyPackets` 函数进行Copy数据

`avutil.CopyPackets(ch.Que, c)`

这里的c为*rtmp.Conn ，也就是函数唯一接收的值。

CopyPackets作用很明显了吧，向ch.Que持续Copy来自Conn的数据。

当然这种Copy是阻塞的，你可以使用goroutine来继续进行下一步操作，也可以就此阻塞，在下方书写释放资源的代码。


这时候有人会问了：

这数据怎么处理呢？目前只写入进频道了，我怎么读？又怎么写？



# 让我访问

由于在此项目中，仅做http和文件方面的读写。其他方法可以仿照参考。

在数据的写入方面，我们只要考虑一件事，从哪来...写到哪去？

## 文件

显然，我们的数据都是从Channel中来的，而且我们要写入到一个新的文件中：

```go

    file, err := os.Create("ovo.flv")

    if err != nil {

        panic(err)

    }

    mux := flv.NewMuxer(file)

    avutil.CopyFile(mux, ch.Que.Oldest())

    defer file.Close()

```

看两个函数

###  flv.NewMuxer

此函数接收一个拥有io.Writer方法的对象，返回一个Muxer

我们将一个文件对象传入即可写入对象

###  avutil.CopyFile

顾名思义，将 ch.Que.Oldest()中的数据写入到Muxer所对应的文件中

其中 ch 为  *Channel       在其他函数中你可能需要先从Channels中获取

`Oldest()` 对应的为 `Latest()`

一个是最新的数据，一个是最前的数据，按需即可。

## HTTP协议

如果你使用的为Gin，则只需传入 `c.Writer` 即可

```go
func GetVideo(c *gin.Context) {

    mix := flv.NewMuxer(c.Writer)

    ch := RtmpServer.Channels[c.Request.URL.Path]

    if ch == nil {

        c.Status(500)

        return

    }
    
    avutil.CopyFile(mix, ch.Que.Latest())

}
```

然后通过其他浏览器或flv播放框架进行读取即可


# 其他

当然，你可能会说，不对啊哥，如果我只进行一个操作呢，那还要创建Channel吗？我看Que的描述是 `One publisher and multiple subscribers thread-safe packet buffer queue.` 啊？

当然，Que只是为了保证线程安全（thread-safe），如果你不需要多线程读取的话，那么只在 `OnPush` 中进行即可

```go
func OnPush(p *rtmp.Conn) {

    file, err := os.Create("test.flv")

    if err != nil {

        panic(err)

    }

    defer file.Close()

    mux := flv.NewMuxer(file)

    avutil.CopyFile(mux, p)

}
```

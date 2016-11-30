[TOC]

# 4.x用户指导

## 前言

刚接触Netty不久, 翻译了一下官网的Netty上手文档. 文档中如果有什么内容和官方存在歧义的话, 请问[官方文档](https://github.com/netty/netty/wiki/User-guide-for-4.x)为准.

如果有什么问题的话, 欢迎随时告知我.

### 问题

如今, 我们使用通用的应用程序或库进行互相通信. 例如, 我们经常使用HTTP客户端库从从Web服务器检索信息; 通过Web服务进行远程过程调用. 可是一个通用协议或者它的实现有时并不能很好的覆盖所有情况, 就像我们不能使用通用协议的HTTP服务进行超大文件传输, 发送邮件, 或者实现像金融信息, 多人游戏等近实时数据交互. 这需要针对特定目的高度优化的协议实现. 比如, 你可以能需要针对基于AJAX的聊天室, 流媒体, 大文件传输进行优化来实现对应的HTTP服务. 另一种不可避免的情况, 你必须处理一个传统的专有协议，以确保与旧系统的互操作性. 这种情况, 在保证原有应用的稳定性和性能的前提下如果快速实现该协议是至关重要的.

### 解决方案

Netty是一个致力于提供异步事件驱动的网络应用框架和工具, 以此进行可维护性, 高性能, 高扩展性协议的服务器或客户端的快速开发.

换句话说, Netty是一个NIO客户端服务器框架, 使网络应用的开发变得更加快速和简单, 例如协议服务器和客户端. 它极大的简化了网络变成, 例如: TCP/UDP套接字服务器开发.

'快速和简单'并不意味着会牺牲最后应用程序的可维护性和性能. Netty的设计借鉴了大量传统协议的经验, 比如: FTP, SMTP, HTTP, 各种二进制和基于文本的协议. 因此, Netty成功找出了一种开发简单并兼顾性能, 稳定性和灵活性的方式.

一些人可能已经发现, 其它的网络框架也声称具有同样的优势. 你可能会问, Netty和它们究竟有什么不同. 答案是以哲学为基础的: 从使用的第一天开始, Netty在API和实现上就会给你最舒适的体验. 这虽然是无法具体描述的, 但是当你读这篇指导并开始使用Netty时, 你会慢慢意识到这种哲学使你的生活更加简单.

## 快速开始

这章主要讲述Netty的核心构建, 通过简单的样例让你快速上手. 当本章结束时, 你便可以以Netty为基础写一个客户端和服务器.

如果你喜欢自上而下的学习, 那你应该从第2章[构建概述]开始, 然后再回到这里.

### 开始之前

运行本章样例的两个最小要求: (1)最新版的Netty (2)JDK1.6+
当你阅读时, 如果对本章涉及到的类有什么问题的话, 可以去查阅[Netty手册](http://netty.io/4.0/api/overview-summary.html)或去[Netty社区](http://netty.io/community.html)问答.

### 写一个Discard服务

世界上最简单的协议不是Hello World而是[DISCARD](http://tools.ietf.org/html/rfc863). 它丢失所有接收到的数据并且没有任何响应.
为了实现`DISCARD`协议, 你唯一要做的就是忽略所有收到的数据. 我们直接从`Handler`的实现开始, 它负责处理Netty生成的I/O事件.

```java
package io.netty.example.discard;

import io.netty.buffer.ByteBuf;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

/**
 * Handles a server-side channel.
 */
public class DiscardServerHandler extends ChannelInboundHandlerAdapter { // (1)

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) { // (2)
        // Discard the received data silently.
        ((ByteBuf) msg).release(); // (3)
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) { // (4)
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}
```

1. `DiscardServerHandler`继承[ChannelInboundHandlerAdapter](http://netty.io/4.0/api/io/netty/channel/ChannelInboundHandlerAdapter.html), [ChannelInboundHandlerAdapter](http://netty.io/4.0/api/io/netty/channel/ChannelInboundHandlerAdapter.html)是[ChannelInboundHandler](http://netty.io/4.0/api/io/netty/channel/ChannelInboundHandler.html)的实现, [ChannelInboundHandler](http://netty.io/4.0/api/io/netty/channel/ChannelInboundHandler.html)提供了很多事件处理方法, 你可以继承并重写它们. 但这个例子里, 直接继承就足够了.
2. 这里我们重写事件处理方法`channelRead()`. 无论什么时候从客户端接收到新数据, 这个方法都会被调用. 这里接收数据的类型是[ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html)
3. 为了实现`DISCARD`协议, 处理器必须忽略所有接收到的数据. [ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html)是一个引用计数对象, 必须通过`release()`方法来明确释放. **请注意: 释放每一个传递进来的引用技术对象是每个`Handler`的职责.** 通常, `channelRead()`方法的实现类似下面这样:

	```java
	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) {
	    try {
	        // Do something with msg
	    } finally {
	        ReferenceCountUtil.release(msg);
	    }
	}
	```
4. 当有异常抛出时, `exceptionCaught()`事件处理函数会被调用. 抛出的异常可能是因为Netty因为I/O错误而抛出, 也可能是因为处理器处理事件而抛出. 在大多数情况下, 这里被捕获的异常应该被记录下来, 同时跟它相关的频道都应该被关闭, 但是该方法的具体实现根据你的具体业务场景而有所不同, 例如: 你可能希望在关闭连接之前返回一个错误码响应.

到目前为止, 一切都还不错. 我们实现了`DISCARD`服务器的前一半. 剩下的是写一个`main()`方法并使用`DiscardServerHandler`启动服务器.

```java
package io.netty.example.discard;

import io.netty.bootstrap.ServerBootstrap;

import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

/**
 * Discards any incoming data.
 */
public class DiscardServer {

    private int port;

    public DiscardServer(int port) {
        this.port = port;
    }

    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap(); // (2)
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class) // (3)
             .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ch.pipeline().addLast(new DiscardServerHandler());
                 }
             })
             .option(ChannelOption.SO_BACKLOG, 128)          // (5)
             .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)

            // Bind and start to accept incoming connections.
            ChannelFuture f = b.bind(port).sync(); // (7)

            // Wait until the server socket is closed.
            // In this example, this does not happen, but you can do that to gracefully
            // shut down your server.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        int port;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        } else {
            port = 8080;
        }
        new DiscardServer(port).run();
    }
}
```

1. [NioEventLoopGroup](http://netty.io/4.0/api/io/netty/channel/nio/NioEventLoopGroup.html)是一个处理I/O操作的多线程事件循环. Netty针对不同的传输提供各种不同的[EventLoopGroup](EventLoopGroup). 在本例中, 我们实现一个服务端应用, 因此使用两个[NioEventLoopGroup](http://netty.io/4.0/api/io/netty/channel/nio/NioEventLoopGroup.html). 第一个我们通常称之为'boss', 它用来接收传入的连接. 第二个通常称之为'worker', 当一个连接被'boss'接收并将其注册到'worker'之后, 'worker'就会处理来自该连接的流量. 使用多少线程以及如何将其映射到已创建的[channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)取决于[EventLoopGroup](http://netty.io/4.0/api/io/netty/channel/EventLoopGroup.html)的实现, 甚至可以通过构造函数来配置.
2. [ServerBootstrap](http://netty.io/4.0/api/io/netty/bootstrap/ServerBootstrap.html)是一个创建服务器的辅助类. 你也可以通过使用[channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)取决于[EventLoopGroup](http://netty.io/4.0/api/io/netty/channel/EventLoopGroup.html)直接创建服务器, 但这是一个很无聊的过程, 大多数情况下你也无须这么做.
3. 本例中我们指定使用[NioServerSocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioServerSocketChannel.html)来实例化一个新的[channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)并用来接收传入的连接.
4. 这里指定的处理器**总是**会被一个新传入的[channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)重新*初始化*[注: 这里的初始化可能并不准确, 但是想不到更好的描述语言]. [ChannelInitializer](http://netty.io/4.0/api/io/netty/channel/ChannelInitializer.html)是一个特殊的处理器, 用于帮助用户配置一个新的[channel](http://netty.io/4.0/api/io/netty/channel/Channel.html). 你很可能需要添加一些处理器来配置这个新[channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)的[ChannelPipeline](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html), 并以此来实现你的网络应用, 比如我们上面所定义的`DiscardServerHandler`. 随着应用程序逐渐复杂, 你可以需要往管道上添加更多的处理器, 以致最终这个抽象类被作为一个独立的顶层类提取出来.
5. 对于具体`Channel`的实现, 你也可以指定一些参数. 我们正在写一个TCP/IP服务器, 所以我们可以设置一些socket选项, 例如: `tcpNoDelay`和`keepAlive`. 关于支持的配置列表可以参考[ChannelOption](http://netty.io/4.0/api/io/netty/channel/ChannelOption.html)的api文档和具体[ChannelConfig](http://netty.io/4.0/api/io/netty/channel/ChannelConfig.html)的实现.
6. 你有注意到`option()`和`childOption()`了吗? `option()`是用来配置[NioServerSocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioServerSocketChannel.html)的, 它用来接收传入的连接. `childOption()`是用来配置[ServerChannel](http://netty.io/4.0/api/io/netty/channel/ServerChannel.html)所接受的子[Channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)的, 在本例中, 子频道是[NioServerSocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioServerSocketChannel.html).
7. 我们已经准备好了, 剩下的就是绑定端口并启动服务. 这里我们为本机的所有网卡绑定8080端口. 另外, 你可以多次调用`bind()`来绑定不同的地址.

恭喜, 你已经使用Netty搭建了第一个服务器.

#### 查看接收到的数据

现在, 我们已经写好了第一个服务, 我们需要测试一下它是否真的正常工作. 最简单的方法是使用`telnet`命令. 例如, 你可以在命令行输入`telnet localhost 8080`并输入一些东西.

可是这就能正面服务正常工作吗? 因为它是一个`DISCARD`服务, 所以我们并不能真的知道. 为了证明它真的在工作, 我们修改一下服务, 把它接收的数据打印出来.

我们已经知道, 无论什么时候收到数据, `channelRead()`都会被调用. 让我们在`DiscardServerHandler`的`channelRead()`方法里加一些代码.

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf in = (ByteBuf) msg;
    try {
        while (in.isReadable()) { // (1)
            System.out.print((char) in.readByte());
            System.out.flush();
        }
    } finally {
        ReferenceCountUtil.release(msg); // (2)
    }
}
```

1. 这个效率低下的循环实际可以简化成如下代码:
	`System.out.println(in.toString(io.netty.util.Charset	Util.US_ASCII))`
2. 另外, 你可以在这里进行`release()`操作.

如果再次你执行`telnet`, 你会发现服务器打印出了它所接收到的数据.

### 写一个Echo服务

目前, 我们已经消费了数据, 但是没有任何响应. 可是一个服务通常都需要响应请求, 现在我们来学习如果通过实现[ECHO](http://tools.ietf.org/html/rfc862)协议, 写一个可以响应消息给客户端的服务.

和上一节实现的discard服务唯一的不同在于我们把收到的数据发送回去, 而不是打印在控制台上. 因为, 只要再次修改`channelRead()`方法就够了:

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
   ctx.write(msg); // (1)
   ctx.flush(); // (2)
}
```

1. [ChannelHandlerContext](http://netty.io/4.0/api/io/netty/channel/ChannelHandlerContext.html)提供很多操作使你能够触发各种各样的I/O事件和操作. 这里我们调用`invoke(Object)`把收到的数据逐字逐句写下来. **注意: 我们并没有想`DISCARD`服务那样释放接收到的数据, 因为当它被写出到wire中时Netty会自动为你释放它**
2. `ctx.write(Object)`并没有让这个消息写出到wire, 而是将其进行内部缓冲, 然后当调用`ctx.flush()`才真正将其写到到wire中. 另外你可以直接调用`ctx.writeAndFlush(msg)`来简化操作.

如果你再次执行`telnet`命令, 你会发现无论你输入什么服务器都会原样返回.

### 写一个Time服务

本部分实现的是[TIME](http://tools.ietf.org/html/rfc868)协议. 它和上面发送消息的例子不同. 它的响应消息包含一个32位的整数, 不接受任何请求数据, 同时响应发送完成之后就立刻关闭连接.

因为我们打算忽略所有接收到的数据, 同时一旦建立连接就发送一个消息, 所以我们这次不能使用`channelRead()`方法, 而是应该重写`channelActive()`方法. 下面是该方法的实现:

```java
package io.netty.example.time;

public class TimeServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(final ChannelHandlerContext ctx) { // (1)
        final ByteBuf time = ctx.alloc().buffer(4); // (2)
        time.writeInt((int) (System.currentTimeMillis() / 1000L + 2208988800L));

        final ChannelFuture f = ctx.writeAndFlush(time); // (3)
        f.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) {
                assert f == future;
                ctx.close();
            }
        }); // (4)
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

1. 正如上面所解释的, 当连接建立并准备产生流量时会调用`channelActive()`方法. 让我们在这个方法里写一个代表当前时间的32位整数.
2. 为了发送一条新消息, 我们需要申请一个新的缓冲区用来存储待发送的消息. 我们打算写一个32位的整数, 所以我们需要一个容量至少4个字节的[ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html). 可以通过`ChannelHandlerContext.alloc()`获取当前[ByteBufAllocator](http://netty.io/4.0/api/io/netty/buffer/ByteBufAllocator.html)并申请一个新的缓冲区.
3. 像往常一样, 我们发送这条消息. 
	
	等等! where's the flip? 在NIO编程中, 在发送一条消息之前难道不需要调用`java.nio.ByteBuffer.flip()`吗? `ByteBuf`并没有这样的方法, 因为它有两个指针: 一个用来进行读操作, 一个用来进行写操作. 当你往`ByteBuf`里写一些东西的时候, 写索引的位置会增加, 同时读索引的位置不变. **读写索引分别代表了当前消息内容的开始位置和结束位置.**
	
	与此Netty相反, 除了可调用的`flip`方法外, NIO缓冲没有提供明确的方法来找出消息内容的开始位置和结束位置. 当你忘记`flip`缓冲区时, 你可以会因为无法发送数据或者无法发送正确的数据而陷入麻烦. 但是在Netty中就不会出现类似问题, 因为我们针对不同的操作使用不同的指针. 当你习惯Netty这种用法之后, 你会发现没有`flip out`的生活如此美好. 
	
	另外需要注意的一点是: `ChannelHandlerContext.write()/ChannelHandlerContext.writeAndFlush`方法返回一个[ChannelFuture](http://netty.io/4.0/api/io/netty/channel/ChannelFuture.html), 每一个[ChannelFuture](http://netty.io/4.0/api/io/netty/channel/ChannelFuture.html)代表一个还未发生的I/O操作. 这意味着, 任何请求操作可能都还没执行, 因为在Netty中所有的操作都是异步的. 例如, 在下面的代码中, 连接可能会在消息发送之前就关闭了.
	
	```java
	Channel ch = ...;
	ch.writeAndFlush(message);
	ch.close();
	```
	
	因此, 你需要在`write()`方法返回的[ChannelFuture](http://netty.io/4.0/api/io/netty/channel/ChannelFuture.html)完成之后调用`close()`方法来通知它的所有监听者: 写操作已经完成了. **请注意: `close()`方法可能也不会立刻关闭连接, 该方法也会返回一个[ChannelFuture](http://netty.io/4.0/api/io/netty/channel/ChannelFuture.html).**
4. 那么当一个写请求结束的时候, 我们怎么得到通知? 只要简单地向返回的`ChannelFuture`中添加一个[ChannelFutureListener](http://netty.io/4.0/api/io/netty/channel/ChannelFutureListener.html)即可. 这里我们创建一个新的匿名[ChannelFutureListener](http://netty.io/4.0/api/io/netty/channel/ChannelFutureListener.html), 它会在操作完成时关闭`channel`
	
	另外, 你可以使用预定义的监听者来简化代码:
	
	```java
	f.addListener(ChannelFutureListener.CLOSE);
	```
	
为了测试我们的Time服务是否按预期工作, 你可以使用UNIX的`rdate`命令:

```sh
$ rdate -o <port> -p <host>
```

这里的host和port是time服务的监听的IP和端口.

### 写一个Time客户端

不像`DISCARD`和`TIME`服务, 我们需要为`TIME`协议写一个客户端, 因为我们无法把32位二进制数据转化为成日历中可识别的日期. 本部分我们学习如何确保服务正常工作以及如何使用Netty写一个客户端.

在Netty中, 客户端和服务器最大的也是仅有的不同是所实现的[Bootstrap](http://netty.io/4.0/api/io/netty/bootstrap/Bootstrap.html)和[Channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)的不同. 如下代码:

```java
package io.netty.example.time;

public class TimeClient {
    public static void main(String[] args) throws Exception {
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap(); // (1)
            b.group(workerGroup); // (2)
            b.channel(NioSocketChannel.class); // (3)
            b.option(ChannelOption.SO_KEEPALIVE, true); // (4)
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new TimeClientHandler());
                }
            });

            // Start the client.
            ChannelFuture f = b.connect(host, port).sync(); // (5)

            // Wait until the connection is closed.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}
```

1. [Bootstrap](http://netty.io/4.0/api/io/netty/bootstrap/Bootstrap.html)和[ServerBootstrap](http://netty.io/4.0/api/io/netty/bootstrap/Bootstrap.html)很类似. 区别在于[Bootstrap](http://netty.io/4.0/api/io/netty/bootstrap/Bootstrap.html)是为非服务器频道服务的, 例如: 客户端频道和无连接频道.
2. 如果你仅指定一个[EventLoopGroup](http://netty.io/4.0/api/io/netty/channel/EventLoopGroup.html), 它会同时被用作boos组和worker组. The boss worker is not used for the client side though.
3. 客户使用[NioSocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioSocketChannel.html)来创建一个[Channel](http://netty.io/4.0/api/io/netty/channel/Channel.html), 而不是使用[NioServerSocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioServerSocketChannel.html).
4. 注意: 这里我们并不像在`ServerBootstrap`中那样使用`childOption()`, 因为客户端`SocketChannel`并没有*父亲*.
5. 我们应该调用`connect()`方法, 而不是`bind()`方法.

正如你所见, 上面的代码和服务端代码并没有什么实质性的不同. 那[ChannelHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html)的实现呢? 它应该从服务器接收一个32位的整数, 并将其转换成我们可读的格式, 然后打印出来, 最后关闭连接. 如下代码:

```java
package io.netty.example.time;

import java.util.Date;

public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf m = (ByteBuf) msg; // (1)
        try {
            long currentTimeMillis = (m.readUnsignedInt() - 2208988800L) * 1000L;
            System.out.println(new Date(currentTimeMillis));
            ctx.close();
        } finally {
            m.release();
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

1. 在TCP/IP中, Netty读取来自peer的数据并将其存入[ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html)中.

上面的代码看起来非常简单, 而且和服务端的例子代码并没有什么不同. 可是这个处理器有时会拒绝工作并抛出一个`IndexOutOfBoundsException`的异常. 我们接下来讨论为什么会出现这个问题.

## 处理基于流的传输

### socket缓冲区的一个小警告

在类似TCP/IP这种基于流的传输协议中, 接收到的数据会被存储到一个socket接收缓冲区中. 不幸的是, 流传输的缓冲区不是一个包队列, 而是一个字节队列. 这意味着, 即使你使用两个独立的包来发送两个独立的消息, 操作系统也不会把它们当作两个独立的消息, 而是把他们当作一堆字节.
因此, 不能保证你所读到的就是你的远程peer所写的.
例如, 我们假设一个操作系统的TCP/IP栈收到下面三个包:

![](https://github.com/lovyhui/netty-guide/blob/master/stream-data-1.png)

由于流传输协议的通用属性, 很大可能你的应用程序会按照下面的零散形式读到它们.

![](https://github.com/lovyhui/netty-guide/blob/master/stream-data-2.png)

因此, 对于接收端, 不管是服务端还是客户端, 都应该对接收到的数据进行碎片整理, 将其整理成一个或多个有意义且容易被应用程序所理解的帧. 对于上面所说的情况, 接收到的数据应该被整理成下面这样:

![](https://github.com/lovyhui/netty-guide/blob/master/stream-data-1.png)

### 第一种解决方法

现在我们回到`TIME`客户端的例子中, 这里我们面临同样的问题. 一个32位的整数是一个非常小的数据, 通常情况下不可能被碎片化. 可是, 它还是可能被碎片化的, 而且这种可能性随着网络流量的增加而增加.

最简单的解决方法是: 建立一个内部累计缓冲区, 等到4个字节的数据全部累计到缓冲区中再进行后续逻辑. 按照这种方式, 修改一下`TimeClientHandler`的实现, 代码如下:


```java
package io.netty.example.time;

import java.util.Date;

public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    private ByteBuf buf;

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) {
        buf = ctx.alloc().buffer(4); // (1)
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) {
        buf.release(); // (1)
        buf = null;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf m = (ByteBuf) msg;
        buf.writeBytes(m); // (2)
        m.release();

        if (buf.readableBytes() >= 4) { // (3)
            long currentTimeMillis = (buf.readUnsignedInt() - 2208988800L) * 1000L;
            System.out.println(new Date(currentTimeMillis));
            ctx.close();
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

1. 一个[ChannelHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html)有两个生命周期监听方法: `handlerAdded()`和`handlerRemoved()`. 你可以在其中执行任意的初始化/去初始化任务, 只要不长时间阻塞即可.
2. 首先, 所有的接收到的数据都应该被累计到`buf`中.
3. 然后, 处理器校验缓冲区中是否有足够的数据, 本例中是4个字节, 然后执行实际的业务逻辑. 否则的话, 当有更多的数据到来时, Netty会继续调用`channelRead()`方法, 直到最后累计足够的数据.

### 第二种解决方法

虽然方法一解决了`TIME`客户端的问题, 但是修改后的代码看起来很不简洁. 想象一个更复杂的协议: 它由多个字段组成, 例如: 变长度字段. 那么, 你的[ChannelInboundHandler](http://netty.io/4.0/api/io/netty/channel/ChannelInboundHandler.html)会很快变得不可维护.

你可能已经注意到了, 你可以向一个[ChannelPipeline](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html)中添加多个[ChannelHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html). 因此, 你可以将单片[ChannelHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html)切分成多个, 进行模块化操作, 以此来降低你的应用程序的复杂度. 例如你可以把`TimeClientHandler`切分成两个处理器:

* `TimeDecoder`用来解决碎片化问题
* `TimeClientHandler`保持原状, 处理业务逻辑

Fortunately, Netty provides an extensible class which helps you write the first one out of the box:
幸运的是, Netty提供了一个可扩展的类来帮助完成上面第一个处理器的编写:

```java
package io.netty.example.time;

public class TimeDecoder extends ByteToMessageDecoder { // (1)
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) { // (2)
        if (in.readableBytes() < 4) {
            return; // (3)
        }

        out.add(in.readBytes(4)); // (4)
    }
}
```

1. [ByteToMessageDecoder](http://netty.io/4.0/api/io/netty/handler/codec/ByteToMessageDecoder.html)实现了[ChannelInboundHandler](http://netty.io/4.0/api/io/netty/channel/ChannelInboundHandler.html),  它可以很方便的解决碎片化问题.
2. 无论什么时候有新的数据到来, [ByteToMessageDecoder](http://netty.io/4.0/api/io/netty/handler/codec/ByteToMessageDecoder.html)都会调用`decode()`方法, 该方法使用一个内部维护的累计缓冲区来缓存接收到的数据.
3. 当累计缓冲区中没有足够的数据时, `decode()`方法不会向`out`中添加数据. 当有新的数据到来时, [ByteToMessageDecoder](http://netty.io/4.0/api/io/netty/handler/codec/ByteToMessageDecoder.html)有再次调用`decode()`方法.
4. 如果`decode()`方法向`out`中添加了一个对象, 意味着改解码器成功解码了一条消息. [ByteToMessageDecoder](http://netty.io/4.0/api/io/netty/handler/codec/ByteToMessageDecoder.html)会忽略累计缓冲区中已读的部分. **注意: 你不需要解码多个消息, [ByteToMessageDecoder](http://netty.io/4.0/api/io/netty/handler/codec/ByteToMessageDecoder.html)会持续调用`decode()`方法, 直至没有数据可以往`out`中添加**

既然我们需要往[ChannelPipeline](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html)中添加另一个处理器, 我们应该修改`TimeClient`中[ChannelInitializer](http://netty.io/4.0/api/io/netty/channel/ChannelInitializer.html)的实现:

```java
b.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(new TimeDecoder(), new TimeClientHandler());
    }
});
```

如果你是一个具有冒险精神的人, 你可能希望尝试使用[ReplayingDecoder](http://netty.io/4.0/api/io/netty/handler/codec/ReplayingDecoder.html), 它更加简化了解码器. 你可以查阅手册获取更多信息.

```java
public class TimeDecoder extends ReplayingDecoder<Void> {
    @Override
    protected void decode(
            ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        out.add(in.readBytes(4));
    }
}
```

此外, Netty提供了`out-of-the-box`解码器, 是你能够很简单的实现多种协议, 帮助你避免单片不可维护的处理器实现所带来的灾难. 请查阅下面的包获取详细样例:

* [io.netty.example.factorial](http://netty.io/4.0/xref/io/netty/example/factorial/package-summary.html) 用于二进制协议
* [io.netty.example.telnet](http://netty.io/4.0/xref/io/netty/example/telnet/package-summary.html) 用于基于行的文本协议

## 使用POJO而不是ByteBuf

目前我们校验过的所有样例都是使用[ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html)作为一个协议消息的主要数据存储结构. 本部分, 我们将使用POJO代替[ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html)来升级`TIME`协议的客户端和服务器.

在[ChannleHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html)中使用POJO的优势很明显: 通过分离从处理器的`ByteBuf`中提取信息的代码, 会使你的代码会变得更容易维护和复用. 在`TIME`的客户端和服务器的例子中, 我们仅仅读取一个32位的整数, 并且直接使用`ByteBuf`也不会有什么大问题. 但是当你实现一个工作中的实际协议时, 你会发现分离提取信息的代码是必须的.

首先, 我们定义一个新的`UnixTime`类型:

```java
package io.netty.example.time;

import java.util.Date;

public class UnixTime {

    private final long value;

    public UnixTime() {
        this(System.currentTimeMillis() / 1000L + 2208988800L);
    }

    public UnixTime(long value) {
        this.value = value;
    }

    public long value() {
        return value;
    }

    @Override
    public String toString() {
        return new Date((value() - 2208988800L) * 1000L).toString();
    }
}
```

现在我们修改`TimeDecoder`, 生产一个`UnixTime`而不是[ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html).

```java
@Override
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    if (in.readableBytes() < 4) {
        return;
    }

    out.add(new UnixTime(in.readUnsignedInt()));
}

```

随着解码器的更新, `TimeClientHandler`也不再使用[ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html).

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    UnixTime m = (UnixTime) msg;
    System.out.println(m);
    ctx.close();
}
```

是不是看起来更加简单和优雅了? 同样, 服务端同样可以使用类似代码. 我们首先更新`TimeServerHandler`:

```java
@Override
public void channelActive(ChannelHandlerContext ctx) {
    ChannelFuture f = ctx.writeAndFlush(new UnixTime());
    f.addListener(ChannelFutureListener.CLOSE);
}
```

现在唯一剩下的是解码器, 它需要实现[ChannelOutboundHandler](http://netty.io/4.0/api/io/netty/channel/ChannelOutboundHandler.html), 并负责将`UnixTime`转换回[ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html). 这笔编写解码器简单的多, 因为编码信息的时候无须考虑包的碎片化和组装的问题.

```java
package io.netty.example.time;

public class TimeEncoder extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        UnixTime m = (UnixTime) msg;
        ByteBuf encoded = ctx.alloc().buffer(4);
        encoded.writeInt((int)m.value());
        ctx.write(encoded, promise); // (1)
    }
}
```

1. 这一行有一些非常重要的事情.

	首先, 我们原样将[ChannelPromise](http://netty.io/4.0/api/io/netty/channel/ChannelPromise.html)传递过去, 当编码数据真正被写到wire中时, Netty会把它标记为成功或失败.
	
	其次, 我们并没有调用`ctx.flush()`. 有一个单独的处理器方法`void flush(ChannelHandlerContext ctx)`, 它的目的是重写`flush()`操作.
	
为了进一步简化操作, 你可以使用[MessageToByteEncoder](http://netty.io/4.0/api/io/netty/handler/codec/MessageToByteEncoder.html):

```java
public class TimeEncoder extends MessageToByteEncoder<UnixTime> {
    @Override
    protected void encode(ChannelHandlerContext ctx, UnixTime msg, ByteBuf out) {
        out.writeInt((int)msg.value());
    }
}
```

最后剩余的工作是在服务端[ChannelPipeline](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html)的`TimeServerHandler`之前插入`TimeEncoder`, 这根本不费什么事.

## 关闭你的应用程序

使用`shutdownGracefully()`关闭一个Netty应用程序, 这就和关闭所有你创建的[EventLoopGroup](http://netty.io/4.0/api/io/netty/channel/EventLoopGroup.html)一样简单, 当所有[EventLoopGroup](http://netty.io/4.0/api/io/netty/channel/EventLoopGroup.html)完全结束并且所有属于它的[Channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)被关闭时, 该函数返回的[Future](http://netty.io/4.0/api/io/netty/util/concurrent/Future.html)会通知你.

## 总结

本章, 我们通过样例快速过了一下如何使用Netty写一个完全工作的网络应用程序.

在即将到来的章节中, 会有更多关于Netty的详细信息. 我们也鼓励你去复查[io.netty.example](https://github.com/netty/netty/tree/4.0/example/src/main/java/io/netty/example)包中的Netty样例.

请注意, [社区](http://netty.io/community.html)时刻等待你的问题和建议, 随时为你提供帮助, 同时Netty和它的文档也会基于你的反馈而不断优化.


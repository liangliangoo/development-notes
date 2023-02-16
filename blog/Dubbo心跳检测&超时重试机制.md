# Dubbo心跳检测%&超时重试机制

源码分析基于Dubbo3.0<a href="https://github.com/liangliangoo/dubbo.git">源码分析仓库地址</a>

## 心跳检测

### Server端的处理方式

dubbo中使用Netty作为网络通信框架，懂netty的话的，看源码会轻松很多

```java

// NettyServer.initServerBootstrap  方法
protected void initServerBootstrap(NettyServerHandler nettyServerHandler) {
  boolean keepalive = getUrl().getParameter(KEEP_ALIVE_KEY, Boolean.FALSE);

  //netty常规配置
  bootstrap.group(bossGroup, workerGroup)
    .channel(NettyEventLoopFactory.serverSocketChannelClass())
    .option(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
    .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
    .childOption(ChannelOption.SO_KEEPALIVE, keepalive)
    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
    // 初始化worker对应的handler
    .childHandler(new ChannelInitializer<SocketChannel>() {
      @Override
      protected void initChannel(SocketChannel ch) throws Exception {
        // FIXME: should we use getTimeout()?
        int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
        if (getUrl().getParameter(SSL_ENABLED_KEY, false)) {
          ch.pipeline().addLast("negotiation", new SslServerTlsHandler(getUrl()));
        }
        ch.pipeline()
          // 编解码handler
          .addLast("decoder", adapter.getDecoder())
          .addLast("encoder", adapter.getEncoder())
          // 添加心跳检测handler
          .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
          // 当心跳检测超时，将会将 IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);  传递给nettyServerHandler处理
          .addLast("handler", nettyServerHandler);
      }
    });
}
```

通过这段源码可以看出，dubbo就是借助netty的IdleStateHandler 处理的心跳检测的，那么接下来就很简单了。

IdleStateHandler 里面一定回去开启定时任务去处理心跳检车，也就是去检测读空闲、写空闲、读写空闲的逻辑，所以需要知道何时开启的定时任务以及定时任务中完成了那些任务。

##### 定时任务的开启时机

```java
// 定时任务初始化方法
private void initialize(ChannelHandlerContext ctx) {
  // Avoid the case where destroy() is called before scheduling timeouts.
  // See: https://github.com/netty/netty/issues/143
  // 初次进入的时候state = 0
  //  状态，0 - 无关， 1 - 初始化完成 2 - 已被销毁
  switch (state) {
    case 1:
    case 2:
      return;
  }

  state = 1;
  initOutputChanged(ctx);

  // 初次进入
  lastReadTime = lastWriteTime = ticksInNanos();
  if (readerIdleTimeNanos > 0) {
    readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                                 readerIdleTimeNanos, TimeUnit.NANOSECONDS);
  }
  if (writerIdleTimeNanos > 0) {
    writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                                 writerIdleTimeNanos, TimeUnit.NANOSECONDS);
  }
  if (allIdleTimeNanos > 0) {
    allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                              allIdleTimeNanos, TimeUnit.NANOSECONDS);
  }
}

// 时机1  在将ChannelHandler添加到实际上下文并准备好处理事件后调用。
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
  if (ctx.channel().isActive() && ctx.channel().isRegistered()) {
    // channelActive() event has been fired already, which means this.channelActive() will
    // not be invoked. We have to initialize here instead.
    initialize(ctx);
  } else {
    // channelActive() event has not been fired yet.  this.channelActive() will be invoked
    // and initialization will occur there.
  }
}


// 时机2    ChannelHandlerContext已向其EventLoop注册
@Override
public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
  // Initialize early if channel is active already.
  if (ctx.channel().isActive()) {
    initialize(ctx);
  }
  super.channelRegistered(ctx);
}

// 时机3    ChannelHandlerContext的通道现在处于活动状态
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
  // This method will be invoked only if this handler was added
  // before channelActive() event is fired.  If a user adds this handler
  // after the channelActive() event, initialize() will be called by beforeAdd().
  initialize(ctx);
  super.channelActive(ctx);
}
```



##### 定时任务做了哪些事情

```java
private final class ReaderIdleTimeoutTask extends AbstractIdleTask {

  ReaderIdleTimeoutTask(ChannelHandlerContext ctx) {
    super(ctx);
  }

  @Override
  protected void run(ChannelHandlerContext ctx) {
    // 自己设置的最大读空闲时间
    long nextDelay = readerIdleTimeNanos;
    // 判断此时是否有读事件发生
    if (!reading) {
      nextDelay -= ticksInNanos() - lastReadTime;
    }

    // 读空闲
    if (nextDelay <= 0) {
      // Reader is idle - set a new timeout and notify the callback.
      // 判断超时的定时任务
      readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);
			
      // 将读空闲事件向下传递
      boolean first = firstReaderIdleEvent;
      firstReaderIdleEvent = false;

      try {
        IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
        channelIdle(ctx, event);
      } catch (Throwable t) {
        ctx.fireExceptionCaught(t);
      }
    } else {
      // Read occurred before the timeout - set a new timeout with shorter delay.
      readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
  }
}
// WriterIdleTimeoutTask   & AllIdleTimeoutTask 所做的逻辑都是大同小异，这里不做分析了
```

##### 当空闲事件闲暇传递以后的处理方式

```java
// NettyServerHandler 处理空闲事件
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
  // server will close channel when server don't receive any heartbeat from client util timeout.
  // 只有Event属于IdleStateEvent 就会关闭对应的channel 并移除cache
  if (evt instanceof IdleStateEvent) {
    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
    try {
      logger.info("IdleStateEvent triggered, close channel " + channel);
      channel.close();
    } finally {
      // 移除缓存
      NettyChannel.removeChannelIfDisconnected(ctx.channel());
    }
  }
  // 向下传递事件
  super.userEventTriggered(ctx, evt);
}
```



### Client端的处理方式

其他逻辑都和server端相同，但是处理空闲事件的handler不同

```java
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
  // send heartbeat when read idle.
  if (evt instanceof IdleStateEvent) {
    try {
      NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
      if (logger.isDebugEnabled()) {
        logger.debug("IdleStateEvent triggered, send heartbeat to channel " + channel);
      }
      // 创建心跳请求报文Request对象
      Request req = new Request();
      req.setVersion(Version.getProtocolVersion());
      req.setTwoWay(true);
      //HEARTBEAT_EVENT表示是心跳报文
      req.setEvent(HEARTBEAT_EVENT);
      // 发送心跳报文
      channel.send(req);
    } finally {
      // 检测当前Channel是否可用，如果不可用则修改状态为非活动状态
      NettyChannel.removeChannelIfDisconnected(ctx.channel());
    }
  } else {
    super.userEventTriggered(ctx, evt);
  }
}
```



### 总结

server和client行为差异主要有两点：

1、当服务端发生超时事件后，服务端会将对应的连接关闭。
2、当客户端发生超时事件后，客户端通过超时重连以及发送心跳尝试维持连接。
主要原因是因为：服务端和客户端对超时后作出的不同操作也反映了双方不同的策略。因为连接占用系统资源，服务端要尽可能的将资源留给其他请求，对于服务端来说，如果某个连接长时间没有数据传输，说明与该客户端的连接已经断开，或者客户端访问已经结束最近不需要再次访问，无论哪种情况，对于服务端来说最好的处理都是断开与客户端的连接。而客户端则不同，客户端想尽全力保证连接的可用，因为客户端访问服务时最希望的是尽快得到响应，因此客户端最好是时时刻刻保持连接的可用，这样访问服务时可以省去建立连接的时间消耗。



## 超时重连

超时重联发生才Client,因为Client希望一直保持长连接，这样可以提高响应速度。当客户端发现某个连接长时间没有收到响应数据，dubbo在exchange信息交换层提供了类HeaderExchangeClient会对该连接进行超时重连。我们来看一下代码，HeaderExchangeClient的构造方法会调用超时重连和心跳检测：

```java
public HeaderExchangeClient(Client client, boolean startTimer) {
  Assert.notNull(client, "Client can't be null");
  this.client = client;
  this.channel = new HeaderExchangeChannel(client);

  if (startTimer) {
    URL url = client.getUrl();
    startReconnectTask(url);
    startHeartBeatTask(url);
  }
}
```



```java
/**
  * 超时重试机制
  * @param url
  */
private void startReconnectTask(URL url) {
  // 可以通过参数“reconnect”设置是否启动重连，默认是true
  if (shouldReconnect(url)) {
    AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);
    // idleTimeout=“heartbeat”*3或者“heartbeat.timeout”，默认空闲超时时间是3分钟
    int idleTimeout = getIdleTimeout(url);
    // heartbeatTimeoutTick=idleTimeout/3，heartbeatTimeoutTick 最小是1000
    long heartbeatTimeoutTick = calculateLeastDuration(idleTimeout);
    // 创建任务 需要剖析任务内容，后面介绍
    ReconnectTimerTask reconnectTimerTask = new ReconnectTimerTask(cp, heartbeatTimeoutTick, idleTimeout);
    // 启动重连任务，每heartbeatTimeoutTick时间执行一次
    reconnectTimer = IDLE_CHECK_TIMER.get().newTimeout(reconnectTimerTask, heartbeatTimeoutTick, TimeUnit.MILLISECONDS);
  }
}
```



ReconnectTimerTask 主要完成的任务：



```java
@Override
public void run(Timeout timeout) throws Exception {
  Collection<Channel> c = channelProvider.getChannels();
  // 遍历连接某一服务端的所有连接
  for (Channel channel : c) {
    if (channel.isClosed()) {
      continue;
    }
    doTask(channel);
  }
  //创建定时任务用于下次检测超时重连，定时任务每次执行完都需要重新创建
  reput(timeout, tick);
}

@Override
protected void doTask(Channel channel) {
  try {
    //获取最后一次收到消息的事件
    Long lastRead = lastRead(channel);
    Long now = now();

    // Rely on reconnect timer to reconnect when AbstractClient.doConnect fails to init the connection
    if (!channel.isConnected()) {
      try {
        logger.info("Initial connection to " + channel);
        ((Client) channel).reconnect();
      } catch (Exception e) {
        logger.error("Fail to connect to " + channel, e);
      }
      // check pong at client
      //如果在指定的时间内没有收到任何消息，则重连，
      //reconnect方法内部有判断，如果当前连接是正常的，则不进行重连
      //这里的idleTimeout是startReconnectTask方法中的heartbeatTimeoutTick，默认是1分钟
    } else if (lastRead != null && now - lastRead > idleTimeout) {
      logger.warn("Reconnect to channel " + channel + ", because heartbeat read idle time out: "
                  + idleTimeout + "ms");
      try {
        ((Client) channel).reconnect();
      } catch (Exception e) {
        logger.error(channel + "reconnect failed during idle time.", e);
      }
    }
  } catch (Throwable t) {
    logger.warn("Exception when reconnect to remote channel " + channel.getRemoteAddress(), t);
  }
}

```


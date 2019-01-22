---
layout:     post
title:      "Dubbo源码分析：Dubbo协议客户端单一长连接下并发调用的结果获取"
subtitle:   "Dubbo Spring"
date:       2018-12-08 22:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Dubbo源码分析
---
### RPC并发调用的结果获取原理
* Dubbo协议在客户端针对每个Service调用，默认是使用单一Netty长连接来处理RPC调用请求的，而在客户端，如在web环境中，任何一个时刻，可能存在多个线程并发对该Service进行并发调用，这些请求都是通过该单一Channel发送和获取结果的，而Netty所有请求都是异步，故dubbo如何保证这些并发线程能正确获取到自己的请求结果，而不会造成数据混乱呢？核心实现为：
1. 客户端Request通过AtomicLong生成的当前进程全局唯一id，服务端响应回传该id；
2. 客户端通过FUTURES静态ConcurrentHashMap保存调用id和异步结果DefaultFuture之间的关系，服务端响应时，查询根据Response的回传请求id，获取该response对应的DefaultFuture，通过await和signal机制实现请求发起线程和结果获取线程之间的通信，最终请求发起线程得到最终的结果。

### 源码实现

* 当客户端发起对服务端的RPC调用时，使用的是DubboInvoker的doInvoker方法：

```java
protected Result doInvoke(final Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
    inv.setAttachment(Constants.VERSION_KEY, version);
    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
        boolean isAsyncFuture = RpcUtils.isGeneratedFuture(inv) || RpcUtils.isFutureReturnType(inv);
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            RpcContext.getContext().setFuture(null);
            return new RpcResult();
        } else if (isAsync) {
            ResponseFuture future = currentClient.request(inv, timeout);
            // For compatibility
            FutureAdapter<Object> futureAdapter = new FutureAdapter<>(future);
            RpcContext.getContext().setFuture(futureAdapter);
            Result result;
            if (isAsyncFuture) {
                // register resultCallback, sometimes we need the asyn result being processed by the filter chain.
                result = new AsyncRpcResult(futureAdapter, futureAdapter.getResultFuture(), false);
            } else {
                result = new SimpleAsyncRpcResult(futureAdapter, futureAdapter.getResultFuture(), false);
            }
            return result;
        } else {
            RpcContext.getContext().setFuture(null);
            return (Result) currentClient.request(inv, timeout).get();
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```
核心关注：调用HeaderExchangeClient发送请求，获取future，这个future是DefaultFuture类，然后封装成FutureAdapter，构造AsyncRpcResult的result：

```java
// 代码1
ResponseFuture future = currentClient.request(inv, timeout);
// For compatibility
FutureAdapter<Object> futureAdapter = new FutureAdapter<>(future);
RpcContext.getContext().setFuture(futureAdapter);
Result result;
if (isAsyncFuture) {
    // register resultCallback, sometimes we need the asyn result being processed by the filter chain.
    result = new AsyncRpcResult(futureAdapter, futureAdapter.getResultFuture(), false);
} else {

AsyncRpcResult的getRpcResult实现：
public Result getRpcResult() {
    Result result;
    try {
        result = resultFuture.get();
    } catch (Exception e) {
        // This should never happen;
        logger.error("", e);
        result = new RpcResult();
    }
    return result;
}
// 即调用了DefaultFuture的get()方法来获取结果，get中会通过DefaultFuture的done，调用done.await进行等待，这里是实现的关键，具体看下面的分析。

// 代码2
// currentClient.request底层最终调用HeaderExchangeChannel的request方法：通过DefaultFuture.newFuture(channel, req, timeout)创建DefaultFuture实例future并返回。

 public ResponseFuture request(Object request, int timeout) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
    }
    // create request.
    Request req = new Request();
    req.setVersion(Version.getProtocolVersion());
    req.setTwoWay(true);
    req.setData(request);
    DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout);
    try {
        channel.send(req);
    } catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}
// 其中Request如下：
public Request() {
    mId = newId();
}
private static long newId() {
    // getAndIncrement() When it grows to MAX_VALUE, it will grow to MIN_VALUE, and the negative can be used as ID
    return INVOKE_ID.getAndIncrement();
}
private static final AtomicLong INVOKE_ID = new AtomicLong(0);
// 这里是关键：INVOKE_ID是静态递增的AtomicLong，即客户端的每次请求都每个请求都是有一个递增唯一的id的，这个id用于在客户端唯一确定一个请求。

// 代码3
DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout)的实现如下：
private DefaultFuture(Channel channel, Request request, int timeout) {
    this.channel = channel;
    this.request = request;
    this.id = request.getId();
    this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
    // put into waiting map.
    FUTURES.put(id, this);
    CHANNELS.put(id, channel);
}
// 其中FUTURES.put(id, this);的FUTURES：
private static final Map<Long, DefaultFuture> FUTURES = new ConcurrentHashMap<>();
// 即为静态常量，存放请求的id和DefaultFuture。

```
* 客户端接收到服务端的RPC调用响应，从底层到顶层依次是NettyClient获取NettyServer的响应，NettyClient将响应向上传递给HeaderExchangeHandler的received方法：

```java
// 代码1
// NettyClient将底层的netty bootstrap交给构造函数传进来的handler处理，这个handler就是HeaderExchangeHandler：
public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
    super(url, wrapChannelHandler(url, handler));
}

@Override
protected void doOpen() throws Throwable {
    NettyHelper.setNettyLoggerFactory();
    bootstrap = new ClientBootstrap(channelFactory);
    // config
    // @see org.jboss.netty.channel.socket.SocketChannelConfig
    bootstrap.setOption("keepAlive", true);
    bootstrap.setOption("tcpNoDelay", true);
    bootstrap.setOption("connectTimeoutMillis", getTimeout());
    final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        @Override
        public ChannelPipeline getPipeline() {
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
            ChannelPipeline pipeline = Channels.pipeline();
            pipeline.addLast("decoder", adapter.getDecoder());
            pipeline.addLast("encoder", adapter.getEncoder());
            pipeline.addLast("handler", nettyHandler);
            return pipeline;
        }
    });
}

// 代码2
// HeaderExchangeHandler的received实现：对于服务端的响应调用handleResponse方法处理
@Override
public void received(Channel channel, Object message) throws RemotingException {
    channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
    final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    try {
        if (message instanceof Request) {
            // handle request.
            Request request = (Request) message;
            if (request.isEvent()) {
                handlerEvent(channel, request);
            } else {
                if (request.isTwoWay()) {
                    handleRequest(exchangeChannel, request);
                } else {
                    handler.received(exchangeChannel, request.getData());
                }
            }
        } else if (message instanceof Response) {
            handleResponse(channel, (Response) message);
        } else if (message instanceof String) {
            if (isClientSide(channel)) {
                Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                logger.error(e.getMessage(), e);
            } else {
                String echo = handler.telnet(channel, (String) message);
                if (echo != null && echo.length() > 0) {
                    channel.send(echo);
                }
            }
        } else {
            handler.received(exchangeChannel, message);
        }
    } finally {
        HeaderExchangeChannel.removeChannelIfDisconnected(channel);
    }
}

// handleResponse的实现：静态方法，通过局部变量，即参数传入的方式保证线程安全，调用DefaultFuture.received方法。
static void handleResponse(Channel channel, Response response) throws RemotingException {
    if (response != null && !response.isHeartbeat()) {
        DefaultFuture.received(channel, response);
    }
}

DefaultFuture.received的实现：
public static void received(Channel channel, Response response) {
    try {
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
            future.doReceived(response);
        } else {
            logger.warn("The timeout response finally returned at "
                    + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                    + ", response " + response
                    + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                    + " -> " + channel.getRemoteAddress()));
        }
    } finally {
        CHANNELS.remove(response.getId());
    }
}

// response将客户端的request的id原样返回了，客户端接收结果线程从FUTURES中移除该请求的id和DefaultFuture实例future，调用future的doReceived处理：调用done的signal通知在done中等待的线程。
private void doReceived(Response res) {
    lock.lock();
    try {
        response = res;
        if (done != null) {
            done.signal();
        }
    } finally {
        lock.unlock();
    }
    if (callback != null) {
        invokeCallback(callback);
    }
}

// 由上面的分析可知，客户端请求时，调用了DefaultFuture的get()方法在请求线程异步来获取结果，get的实现如下：在done调用await等待结果，从而通过await和signal实现线程之间的通信，客户端请求线程得到通知最终获取到了结果。
@Override
public Object get(int timeout) throws RemotingException {
    if (timeout <= 0) {
        timeout = Constants.DEFAULT_TIMEOUT;
    }
    if (!isDone()) {
        long start = System.currentTimeMillis();
        lock.lock();
        try {
            while (!isDone()) {
                done.await(timeout, TimeUnit.MILLISECONDS);
                if (isDone() || System.currentTimeMillis() - start > timeout) {
                    break;
                }
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            lock.unlock();
        }
        if (!isDone()) {
            throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
        }
    }
    return returnFromResponse();
}

```
* 服务端从Netty Server接收请求，然后向上传给HeaderExchangeHandler处理：

```java
void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
    Response res = new Response(req.getId(), req.getVersion());
    if (req.isBroken()) {
        Object data = req.getData();

        String msg;
        if (data == null) msg = null;
        else if (data instanceof Throwable) msg = StringUtils.toString((Throwable) data);
        else msg = data.toString();
        res.setErrorMessage("Fail to decode request due to: " + msg);
        res.setStatus(Response.BAD_REQUEST);

        channel.send(res);
        return;
    }
    // find handler by message class.
    Object msg = req.getData();
    try {
        // handle data.
        CompletableFuture<Object> future = handler.reply(channel, msg);
        if (future.isDone()) {
            res.setStatus(Response.OK);
            res.setResult(future.get());
            channel.send(res);
            return;
        }
        future.whenComplete((result, t) -> {
            try {
                if (t == null) {
                    res.setStatus(Response.OK);
                    res.setResult(result);
                } else {
                    res.setStatus(Response.SERVICE_ERROR);
                    res.setErrorMessage(StringUtils.toString(t));
                }
                channel.send(res);
            } catch (RemotingException e) {
                logger.warn("Send result to consumer failed, channel is " + channel + ", msg is " + e);
            } finally {
                // HeaderExchangeChannel.removeChannelIfDisconnected(channel);
            }
        });
    } catch (Throwable e) {
        res.setStatus(Response.SERVICE_ERROR);
        res.setErrorMessage(StringUtils.toString(e));
        channel.send(res);
    }
}

// 构造response，获取客户端请求req的id，进行回传：Response res = new Response(req.getId(), req.getVersion());

// handle data.
CompletableFuture<Object> future = handler.reply(channel, msg);
if (future.isDone()) {
    res.setStatus(Response.OK);
    res.setResult(future.get());
    channel.send(res);
    return;
}
// 调用handler.reply，最终调用本地的Service，进行方法调用，即我们在配置文件中指定的dubbo:service的ref参数对应的bean。
// handler是DubboProtocol中的requestHandler：
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
    @Override
    public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {
        if (message instanceof Invocation) {
            Invocation inv = (Invocation) message;
            Invoker<?> invoker = getInvoker(channel, inv);
            // need to consider backward-compatibility if it's a callback
            if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                String methodsStr = invoker.getUrl().getParameters().get("methods");
                boolean hasMethod = false;
                if (methodsStr == null || !methodsStr.contains(",")) {
                    hasMethod = inv.getMethodName().equals(methodsStr);
                } else {
                    String[] methods = methodsStr.split(",");
                    for (String method : methods) {
                        if (inv.getMethodName().equals(method)) {
                            hasMethod = true;
                            break;
                        }
                    }
                }
                if (!hasMethod) {
                    logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                            + " not found in callback service interface ,invoke will be ignored."
                            + " please update the api interface. url is:"
                            + invoker.getUrl()) + " ,invocation is :" + inv);
                    return null;
                }
            }
            RpcContext rpcContext = RpcContext.getContext();
            boolean supportServerAsync = invoker.getUrl().getMethodParameter(inv.getMethodName(), Constants.ASYNC_KEY, false);
            if (supportServerAsync) {
                CompletableFuture<Object> future = new CompletableFuture<>();
                rpcContext.setAsyncContext(new AsyncContextImpl(future));
            }
            rpcContext.setRemoteAddress(channel.getRemoteAddress());
            Result result = invoker.invoke(inv);

            if (result instanceof AsyncRpcResult) {
                return ((AsyncRpcResult) result).getResultFuture().thenApply(r -> (Object) r);
            } else {
                return CompletableFuture.completedFuture(result);
            }
        }
        throw new RemotingException(channel, "Unsupported request: "
                + (message == null ? null : (message.getClass().getName() + ": " + message))
                + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
    }
 ...
}
// 核心为：invoker在ServiceConfig的export时，封装了实际Server的ref，invoke最终交给ref进行方法调用。
Invoker<?> invoker = getInvoker(channel, inv);
Result result = invoker.invoke(inv);

```
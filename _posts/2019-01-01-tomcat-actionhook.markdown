---
layout:     post
title:      "Tomcat源码分析：ActionHook回调机制"
subtitle:   "Tomcat ActionHook"
date:       2019-01-01 22:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Tomcat源码分析
---
### 回调机制
* ActionHook：servlet容器到应用层协议处理器processor的回调机制。具体为servlet容器通过连接器connector，实现到应用层协议处理器（具体为coyote包的ProtocolHandler的Processor）的回调，作用是通过应用层协议处理器Processor来对servlet容器产生的response进行加工，添加特定应用层协议相关的数据，以及将数据写出到底层socket。
### 设计思路
1. Connector通过包含一个ProtocolHandler的引用来和应用层协议实现建立联系，如http1.1应用层协议将封装好的http1.1格式的request和response，（具体为通过CoyoteAdapter）传给Connector，然后Connector传给Container。但是反过来Container请求处理完成之后，进行响应时，可能需要根据使用的应用层协议，如http1.1，对Response进行加工，还有就是将response通过底层socket进行写出，而ProtocolHandler是应用层协议的实现，以及和底层socket通过一个Endpoint引用进行关联，故需要一种机制来实现servlet容器Container到ProtocolHandler的协议处理器的调用。
2. 但是为了降低整体架构的耦合和整体架构设计的职责单一，不能直接在Container中调用ProtocolHandler的协议处理器Processor。
3. 同时，由于请求接收时，ProtocolHandler是通过Connector来将应用层数据（具体为通过CoyoteAdapter封装后）传给Container，故可以反过来，即Container可以通过连接器（具体为通过connector包的CoyoteOutputStream，往下依次传递到coyote包的Response，Response的OutputBuffer，OutputBuffer的Processor）来回调ProtocolHandler（具体为coyote包的Processor），而ActionHook就是这个回调的设计和实现，即coyote包的Processor会实现ActionHook接口并实现action方法。
4. ActionHook通过ActionCode机制来进行实际的回调处理。

### 案例：DefaultServlet
1. 静态资源的请求一般通过DefaultServlet来处理。如下为doGet调用的serveResource来进行资源获取和响应：

```
 /**
 * Serve the specified resource, optionally including the data content.
 *
 * @param request  The servlet request we are processing
 * @param response The servlet response we are creating
 * @param content  Should the content be included?
 * @param encoding The encoding to use if it is necessary to access the
 *                 source as characters rather than as bytes
 *
 * @exception IOException if an input/output error occurs
 * @exception ServletException if a servlet-specified error occurs
 */
protected void serveResource(HttpServletRequest request,
                             HttpServletResponse response,
                             boolean content,
                             String encoding)
    throws IOException, ServletException {
}    
```

2. serveResource底层会调用copyRange来将数据写出：具体为ostream.write，其中ostream的实现为connector包的CoyoteOutputStream。ostream是从connector包的Response创建的。
```
protected IOException copyRange(InputStream istream,
                                  ServletOutputStream ostream,
                                  long start, long end) {

    ...
    
    while ( (bytesToRead > 0) && (len >= buffer.length)) {
        try {
            len = istream.read(buffer);
            if (bytesToRead >= len) {
                ostream.write(buffer, 0, len);
                bytesToRead -= len;
            } else {
                ostream.write(buffer, 0, (int) bytesToRead);
                bytesToRead = 0;
            }
        } catch (IOException e) {
            exception = e;
            len = -1;
        }
        if (len < buffer.length)
            break;
    }

    return exception;

}
```
3. CoyoteOutputStream的write方法实现：底层调用基类OutputBuffer的write，一直往下最终则是调用了coyote的Response的doWrite。此时在代码结构层面，则是完成了connector包到coyote包的过渡。

```
@Override
public void write(byte[] b) throws IOException {
    write(b, 0, b.length);
}


@Override
public void write(byte[] b, int off, int len) throws IOException {
    boolean nonBlocking = checkNonBlockingWrite();
    ob.write(b, off, len);
    if (nonBlocking) {
        checkRegisterForWrite();
    }
}
```

4. coyote的Response的doWrite，doWrite最终调用的是coyote的OutPutBuffer的doWrite：

```
/**
 * Write a chunk of bytes.
 *
 * @param chunk The ByteBuffer to write
 *
 * @throws IOException If an I/O error occurs during the write
 */
public void doWrite(ByteBuffer chunk) throws IOException {
    int len = chunk.remaining();
    outputBuffer.doWrite(chunk);
    contentWritten += len - chunk.remaining();
}
```
5. coyote的OutputBuffer则是根据具体的应用层协议来实现doWrite，以下为Http11OutputBuffer的实现：

```
@Override
public int doWrite(ByteBuffer chunk) throws IOException {

    if (!response.isCommitted()) {
        // Send the connector a request for commit. The connector should
        // then validate the headers, send them (using sendHeaders) and
        // set the filters accordingly.
        response.action(ActionCode.COMMIT, null);
    }

    if (lastActiveFilter == -1) {
        return outputStreamOutputBuffer.doWrite(chunk);
    } else {
        return activeFilters[lastActiveFilter].doWrite(chunk);
    }
}
```

6. 由5可知:
* 先调用了repsonse.action(ActionCode.COMMIT, null)，这个地方就是调用了ActionHook接口的action回调方法了，作用是如注释所示：对response的http报文的 header进行检查和设值，而这个是跟应用层协议相关的，故需要回到coyote包的ProtocolHandler的Processor这边来处理；以及添加相关过滤器OutputFilter，如压缩GzipOutputFilter。此处为Http11Processor提供了ActionHook的action的实现。

* 处理完成之后，则直接写出到socket或者如果有OutputFilter，则交个filter过滤器链处理完之后在写出到socket。此处为通过Http11OutputBuffer的内部类SocketOutputBuffer来完成到socket的写出。

```
/**
 * This class is an output buffer which will write data to a socket.
 */
protected class SocketOutputBuffer implements HttpOutputBuffer {

    /**
     * Write chunk.
     */
    @Override
    public int doWrite(ByteBuffer chunk) throws IOException {
        try {
            int len = chunk.remaining();
            socketWrapper.write(isBlocking(), chunk);
            len -= chunk.remaining();
            byteCount += len;
            return len;
        } catch (IOException ioe) {
            response.action(ActionCode.CLOSE_NOW, ioe);
            // Re-throw
            throw ioe;
        }
    }
}
```
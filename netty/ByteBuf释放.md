# ByteBuf简述
ByteBuf总的来说包含两大类：Pooled和UnPooled。netty自4.0之后，引入了ReferenceCounted接口，来管理Buffer资源。官方是建议手动来释放ByteBuf，释放之后，ByteBuf要么返回到Pool中(Pool类型)，要么立即释放(UnPooled类型)。
	
# 使用事例：
官网中列出了若干试用方法
	
```java
ByteBufAllocator alloc = PooledByteBufAllocator.DEFAULT;
ByteBuf buf = alloc.directBuffer(1024);
...
buf.release(); // The direct buffer is returned to the pool.
```
	
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
	ByteBuf buf = (ByteBuf) msg;
		try {
				...
		} finally {
			buf.release();
		}
	}
```
	
需要注意的是：释放ByteBuf必须遵守一定的规则

1. 首先必须是InBound类型的handler。
2. 必须是最后一个试用ByteBuf的Handler。如果提前释放了ByteBuf，后续的Handler是无法使用数据的。
3. Outbound类型的handler没有必要显示释放ByteBuf，因为netty内部自己完成了。
	
# 本人使用过程中遇到的问题

调研了一天netty如何管理ByteBuf，为了防止应用内存泄漏。按照官方的建议，释放ByteBuf。结果抛出如下异常：
	
```java
io.netty.util.IllegalReferenceCountException: refCnt: 0, decrement: 1
at io.netty.buffer.AbstractReferenceCountedByteBuf.release(AbstractReferenceCountedByteBuf.java:111)
at io.netty.handler.codec.ByteToMessageDecoder.channelRead(ByteToMessageDecoder.java:256)
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:372)
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:358)
at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:350)
at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1334)
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:372)
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:358)
at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:926)
at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:129)
at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:610)
at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:551)
at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:465)
at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:437)
at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:873)
at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:144)
at java.lang.Thread.run(Thread.java:745)
io.netty.util.IllegalReferenceCountException: refCnt: 0, decrement: 1
at io.netty.buffer.AbstractReferenceCountedByteBuf.release(AbstractReferenceCountedByteBuf.java:111)
at io.netty.handler.codec.ByteToMessageDecoder.channelInputClosed(ByteToMessageDecoder.java:350)
at io.netty.handler.codec.ByteToMessageDecoder.channelInactive(ByteToMessageDecoder.java:325)
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelInactive(AbstractChannelHandlerContext.java:255)
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelInactive(AbstractChannelHandlerContext.java:241)
at io.netty.channel.AbstractChannelHandlerContext.fireChannelInactive(AbstractChannelHandlerContext.java:234)
at io.netty.channel.DefaultChannelPipeline$HeadContext.channelInactive(DefaultChannelPipeline.java:1329)
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelInactive(AbstractChannelHandlerContext.java:255)
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelInactive(AbstractChannelHandlerContext.java:241)
at io.netty.channel.DefaultChannelPipeline.fireChannelInactive(DefaultChannelPipeline.java:908)
at io.netty.channel.AbstractChannel$AbstractUnsafe$7.run(AbstractChannel.java:744)
at io.netty.util.concurrent.AbstractEventExecutor.safeExecute(AbstractEventExecutor.java:163)
at io.netty.util.concurrent.SingleThreadEventExecutor.runAllTasks(SingleThreadEventExecutor.java:418)
at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:440)
at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:873)
at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:144)
at java.lang.Thread.run(Thread.java:745)
```
	
本人写的代码片段如下：
```java
ByteBuf buf = (ByteBuf) msg;
byte[] bytes = new byte[buf.readableBytes()];
buf.readBytes(bytes);
				
try {
	bufferedOutputStream.write(bytes, 0, bytes.length);
	size = size + bytes.length;
					
	if (size == request.getFileSize()) {//传输完成
		bufferedOutputStream.close();
						
		TransferHelper.decompress(compressedFile, request.getDecompressPath());
		TransferResponse response = constructResponse(TransferResponse.SUCCESS, TransferResponse.END);
		ctx.writeAndFlush(response);
		ctx.close();
	}
					
} catch (Throwable e) {
	e.printStackTrace();
	throw new ProcessingException(e);
} finally {
	ReferenceCountUtil.release(buf);
	buf.release();
}
```
网上检索了下，没有任何线索，老办法直接深入源码，在ByteToMessageDecoder这个类中：
	
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
  if (msg instanceof ByteBuf) {
		CodecOutputList out = CodecOutputList.newInstance();
		try {
			ByteBuf data = (ByteBuf) msg;
			first = cumulation == null;
			if (first) {
				cumulation = data;
			} else {
				cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
			}
			callDecode(ctx, cumulation, out);
		} catch (DecoderException e) {
			 throw e;
		} catch (Throwable t) {
			throw new DecoderException(t);
		} finally {
			if (cumulation != null && !cumulation.isReadable()) {
				numReads = 0;
				cumulation.release();
				cumulation = null;
			} else if (++ numReads >= discardAfterReads) {
				 // We did enough reads already try to discard some bytes so we not risk to see a OOME.
				 // See https://github.com/netty/netty/issues/4275
				 numReads = 0;
				 discardSomeReadBytes();
			}

			int size = out.size();
			decodeWasNull = !out.insertSinceRecycled();
			fireChannelRead(ctx, out, size);
			out.recycle();
		}
 } else {
    ctx.fireChannelRead(msg);
 }
}
	
```
	
内部也释放ByteBuf。
	
## 问题原因
我在最外层handler释放了ByteBuf，但是在下层的ByteToMessageDecoder中，也要释放ByteBuf，当检查到已经释放ByteBuf后，就抛出异常：AbstractReferenceCountedByteBuf类：
	
```java
public boolean release() {
  for (;;) {
    int refCnt = this.refCnt;
    if (refCnt == 0) {
       throw new IllegalReferenceCountException(0, -1);
    }

    if (refCntUpdater.compareAndSet(this, refCnt, refCnt - 1)) {
      if (refCnt == 1) {
        deallocate();
        return true;
      }
      return false;
    }
  }
}
```
	
#结论
netty 有些自带的handler已经帮我们释放了ByteBuf，这种情况下，我们没有必要再手动释放，否则就抛出异常了，如果netty自带的handler没有帮我们释放ByteBuf，那么还是需要手动释放，来增加整体性能。（针对Pool类型的ByteBuf，释放后，直接返回Pool中，Unpooled类型的ByteBuf，手动释放可能防止内存泄漏）

#参考资料
* [Reference counted objects](http://netty.io/wiki/reference-counted-objects.html)
* [Using as a generic library](http://netty.io/wiki/using-as-a-generic-library.html)
* Netty In Action


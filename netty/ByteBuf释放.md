# ByteBuf����
ByteBuf�ܵ���˵���������ࣺPooled��UnPooled��netty��4.0֮��������ReferenceCounted�ӿڣ�������Buffer��Դ���ٷ��ǽ����ֶ����ͷ�ByteBuf���ͷ�֮��ByteBufҪô���ص�Pool��(Pool����)��Ҫô�����ͷ�(UnPooled����)��
	
# ʹ��������
�������г����������÷���
	
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
	
��Ҫע����ǣ��ͷ�ByteBuf��������һ���Ĺ���

1. ���ȱ�����InBound���͵�handler��
2. ���������һ������ByteBuf��Handler�������ǰ�ͷ���ByteBuf��������Handler���޷�ʹ�����ݵġ�
3. Outbound���͵�handlerû�б�Ҫ��ʾ�ͷ�ByteBuf����Ϊnetty�ڲ��Լ�����ˡ�
	
# ����ʹ�ù���������������

������һ��netty��ι���ByteBuf��Ϊ�˷�ֹӦ���ڴ�й©�����չٷ��Ľ��飬�ͷ�ByteBuf������׳������쳣��
	
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
	
����д�Ĵ���Ƭ�����£�
```java
ByteBuf buf = (ByteBuf) msg;
byte[] bytes = new byte[buf.readableBytes()];
buf.readBytes(bytes);
				
try {
	bufferedOutputStream.write(bytes, 0, bytes.length);
	size = size + bytes.length;
					
	if (size == request.getFileSize()) {//�������
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
���ϼ������£�û���κ��������ϰ취ֱ������Դ�룬��ByteToMessageDecoder������У�
	
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
	
�ڲ�Ҳ�ͷ�ByteBuf��
	
## ����ԭ��
���������handler�ͷ���ByteBuf���������²��ByteToMessageDecoder�У�ҲҪ�ͷ�ByteBuf������鵽�Ѿ��ͷ�ByteBuf�󣬾��׳��쳣��AbstractReferenceCountedByteBuf�ࣺ
	
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
	
#����
netty ��Щ�Դ���handler�Ѿ��������ͷ���ByteBuf����������£�����û�б�Ҫ���ֶ��ͷţ�������׳��쳣�ˣ����netty�Դ���handlerû�а������ͷ�ByteBuf����ô������Ҫ�ֶ��ͷţ��������������ܡ������Pool���͵�ByteBuf���ͷź�ֱ�ӷ���Pool�У�Unpooled���͵�ByteBuf���ֶ��ͷſ��ܷ�ֹ�ڴ�й©��

#�ο�����
* [Reference counted objects](http://netty.io/wiki/reference-counted-objects.html)
* [Using as a generic library](http://netty.io/wiki/using-as-a-generic-library.html)
* Netty In Action


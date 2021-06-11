# <font color=#3da742>SocketChannel</font>

## <font color=#3da742>SocketChannel的创建方式： </font>

```java
try{
    SocketChannel instance = SocketChannel.open();
}
catch(IOException e){
    // TODO doSomething
}
```



* 不可能为任意的、预先存在的socket创建通道
* 一个刚新创建的socket channel的状态是：新建但是没有连接，任何尝试在一个未连接的socket channel上的IO操作都会抛出NotYetConnectedException异常。
* 可以调用connect函数来为socket channel建立连接
* 直到socket channel被关闭之前，它一直处于已连接状态，可以通过函数isConnected来检测它的连接状态

## <font color=#3da742>Socket Channel连接支持非阻塞式连接</font>

一个Socket channel被创建后，之后连接到远程socket可以通过connect函数来进行，最后调用finishConnect函数来完成连接。判断一个连接操作是否正在执行中可以通过isConnectionPending函数来检测。

## <font color=#3da742>Socket Channel支持异步关闭</font>

对于Socket的输入端而言，有一读线程(记做$T_1$)对Socket Channel中的数据进行读操作，此时另一线程$T_2$将Socket关闭了，那么对于读线程$T_1$来说，它在Channel上的读操作就会发生阻塞，并发生返回。从而发生$T_1$线程的异步关闭。同理，对于Socket的输出端，写操作也会发生异步关闭的情况。

## <font color=#3da742>类</font>

```java
public abstract class SocketChannel
    extends AbstractSelectableChannel
    implements ByteChannel,ScatteringByteChannel,GatheringByteChannel,NetworkChannel{
    
    // 初始化实例对象
    protected SocketChannel(SelectorProvider provider){
        super(provider)
    }
    
    // 打开一个socket channel
    public static SocketChannel open() throws IOException{
        return SelectorProvider.provider().openSocketChannel();
    }
    
    // 打卡一个socket channel,并将其连接到一个远程地址
    public static SocketChannel open(SocketAddress remote) throws IOException{
        SocketChannel sc = open();
        try{
            sc.connect(remote);
        } catch (Throwable x) {
            try{
                sc.close()
            } catch (Throwable suppressed) {
                x.addSuppressed(suppressed);
            }
            throw x;
        }
        assert sc.isConnected();
        return sc;
    }
    
    public final int validOps(){
        return (SelectionKey.OP_READ | SelectionKey.OP_WRITE | SelectionKey.OP_CONNECT);
    }
    
    public abstract SocketChannel bind(SocketAddress local) throws IOException;
    
    public abstract <T> SocketChannel setOption(SocketOption<T> name,T value) throws IOException;
    
    public abstract SocketChannel shutdownInput() throws IOException;
    
    // 返回和当前Socket Channel连接的Socket
    public abstract Socket socket();
    
    // 判断当前Socket Channel是否处于连接状态
    public abstract boolean isConnected();
    
    // 判断当前Socket Channel的连接过程是否在执行(详细情况参照"Socket Channel连接支持非阻塞式连接")
    public abstract boolean isConnectionPending();
    
    // 可以通过抛出的异常子类来知道发生了何种异常信息
    public abstract boolean connect(SocketAddress remote) throws IOException;
    
    // 关闭连接到一个socket channel的过程
    public abstract boolean finishConnect() throws IOException;
    
    // 获取当前Socket Channel连接的远程地址
    public abstract SocketAddress getRemoteAddress() throws IOException;
    
    // 从输入Buffer中读取远程传来的数据
    public abstract int read(ByteBuffer dst) throws IOException;
    
    // 同上
    public abstract long read(ByteBuffer[] dsts,int offset,int length) throws IOException;
    
    // 同上
    public final long read(ByteBuffer[] dsts) throws IOException{
        return read(dsts,0,dsts.length);
    }
    // 将数据写到Buffer以便于后续发送到远程
    public abstract int write(ByteBuffer src) throws IOException;
    
    // 同上
    public abstract long write(ByteBuffer[] srcs,int offset,int length) throws IOException;
    
    // 同上
    public final long write(ByteBuffer[] srcs) throws IOException{
        write(srcs,0,srcs.length);
    }
    
    // 先不去管它的作用，用到再说
    public abstract SocketAddress getLocalAddress() throws IOException;
}
```


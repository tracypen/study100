### 一、IO模型

#### 1.BIO模型

```java
public class IOServer {

    /**
     * Server服务端首先创建ServerSocket监听8000端口,然后创建线程不断调用阻塞方法 serversocket.accept()获取新的连接,当获取到新的连接给每条连接创建新的线程负责从该连接中读取数据,然后读取数据是以字节流的方式
     *
     * @param args
     * @throws IOException
     */
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8000);

        //接收新连接线程
        new Thread(() -> {
            try {
                //(1)阻塞方法获取新的连接
                Socket socket = serverSocket.accept();

                //(2)每一个新的连接都创建一个线程,负责读取数据
                new Thread(() -> {
                    try {
                        byte[] data = new byte[1024];
                        InputStream inputStream = socket.getInputStream();
                        while (true) {
                            int len;
                            //(3)按照字节流方式读取数据
                            while ((len = inputStream.read(data)) != -1)
                                System.out.println(new String(data, 0, len));
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }).start();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
public class IOClient {

    /**
     * Client客户端连接服务端8000端口每隔2秒向服务端写带有时间戳的 "hello world"
     *
     * @param args
     */
    public static void main(String[] args) {
        new Thread(() -> {
            try {
                Socket socket = new Socket("127.0.0.1", 8000);
                while (true) {
                    try {
                        socket.getOutputStream().write((new Date() + ": hello world").getBytes());
                        socket.getOutputStream().flush();
                        Thread.sleep(2000);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

传统的IO模型每个连接创建成功都需要一个线程来维护,每个线程包含一个while死循环,那么1w个连接对应1w个线程,继而1w个while死循环带来如下几个问题:

1. 线程资源受限:线程是操作系统中非常宝贵的资源,同一时刻有大量的线程处于阻塞状态是非常严重的资源浪费,操作系统耗不起;

2. 线程切换效率低下:单机CPU核数固定,线程爆炸之后操作系统频繁进行线程切换,应用性能急剧下降;
3. 数据读写是以字节流为单位效率不高:每次都是从操作系统底层一个字节一个字节地读取数据.



#### 2.NIO

1.线程资源受限:NIO编程模型新来一个连接不再创建一个新的线程,把这条连接直接绑定到某个固定的线程,然后这条连接所有的读写都由该线程来负责.把这么多while死循环变成一个死循环,这个死循环由一个线程控制,一条连接来了,不创建一个while死循环去监听是否有数据可读,直接把这条连接注册到Selector上,然后通过检查Selector批量监测出有数据可读的连接进而读取数据.

2. 线程切换效率低下:线程数量大大降低,线程切换效率因此也大幅度提高.
3. 数据读写是以字节流为单位效率不高:NIO维护一个缓冲区每次从这个缓冲区里面读取一块的数据,数据读写不再以字节为单位,而是以字节块为单位.

```java
public class NIOServer {

    /**
     * serverSelector负责轮询是否有新的连接,clientSelector负责轮询连接是否有数据可读.
     * 服务端监测到新的连接不再创建一个新的线程,而是直接将新连接绑定到clientSelector上,这样不用IO模型中1w个while循环在死等
     * clientSelector被一个while死循环包裹,如果在某一时刻有多条连接有数据可读通过 clientSelector.select(1)方法轮询出来进而批量处理
     * 数据的读写以内存块为单位
     *
     * @param args
     * @throws IOException
     */
    public static void main(String[] args) throws IOException {
        Selector serverSelector = Selector.open();
        Selector clientSelector = Selector.open();

        new Thread(() -> {
            try {
                ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
                serverSocketChannel.socket().bind(new InetSocketAddress(8000));
                serverSocketChannel.configureBlocking(false);
                serverSocketChannel.register(serverSelector, SelectionKey.OP_ACCEPT);

                while (true) {
                    // 轮询监测是否有新的连接
                    if (serverSelector.select(1) > 0) {
                        Set<SelectionKey> selectionKeys = serverSelector.selectedKeys();
                        Iterator<SelectionKey> keyIterator = selectionKeys.iterator();
                        while (keyIterator.hasNext()) {
                            SelectionKey selectionKey = keyIterator.next();
                            if (selectionKey.isAcceptable()) {
                                try {
                                    //(1)每来一个新连接不需要创建一个线程而是直接注册到clientSelector
                                    SocketChannel socketChannel = ((ServerSocketChannel) selectionKey.channel()).accept();
                                    socketChannel.configureBlocking(false);
                                    socketChannel.register(clientSelector, SelectionKey.OP_READ);
                                } finally {
                                    keyIterator.remove();
                                }
                            }
                        }
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();

        new Thread(() -> {
            try {
                while (true) {
                    // (2)批量轮询是否有哪些连接有数据可读
                    if (clientSelector.select(1) > 0) {
                        Set<SelectionKey> selectionKeys = serverSelector.selectedKeys();
                        Iterator<SelectionKey> keyIterator = selectionKeys.iterator();
                        while (keyIterator.hasNext()) {
                            SelectionKey selectionKey = keyIterator.next();
                            if (selectionKey.isReadable()) {
                                try {
                                    SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
                                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                                    //(3)读取数据以块为单位批量读取
                                    socketChannel.read(byteBuffer);
                                    byteBuffer.flip();
                                    System.out.println(Charset.defaultCharset().newDecoder().decode(byteBuffer)
                                            .toString());
                                } finally {
                                    keyIterator.remove();
                                    selectionKey.interestOps(SelectionKey.OP_READ);
                                }
                            }
                        }
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

#### 3.Netty

```java
public class NettyServer {

    public static void main(String[] args) {
        ServerBootstrap serverBootstrap = new ServerBootstrap();

        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();

        serverBootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new StringDecoder());
                        ch.pipeline().addLast(new SimpleChannelInboundHandler<String>() {
                            @Override
                            protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
                                System.out.println(msg);
                            }
                        });
                    }
                }).bind(8000);
    }
}
public class NettyClient {

    public static void main(String[] args) throws InterruptedException {
        Bootstrap bootstrap = new Bootstrap();
        NioEventLoopGroup group = new NioEventLoopGroup();

        bootstrap.group(group).channel(NioSocketChannel.class).handler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                ch.pipeline().addLast(new StringEncoder());
            }
        });

        Channel channel = bootstrap.connect("127.0.0.1", 8000).channel();
        while (true) {
            channel.writeAndFlush(new Date() + ": hello world!");
            Thread.sleep(2000);
        }
    }
}
```

### 二、启动流程

#### 1.服务端启动流程

![](https://upload-images.jianshu.io/upload_images/8387919-c55b718189f1b0a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```java
/**
 * 01: 服务端启动流程介绍[https://www.jianshu.com/p/ec3ebb396943]
 * 要启动Netty服务端,必须要指定三类属性,分别是线程模型、IO模型、连接读写处理逻辑
 * Netty服务端启动的流程是创建引导类给引导类指定线程模型,IO模型,连接读写处理逻辑,绑定端口之后服务端就启动起来
 * bind方法是异步的通过异步机制来实现端口递增绑定
 * Netty服务端启动额外的参数,主要包括给服务端channel或者channel设置属性值,设置底层TCP参数
 */
public class NettyServer {

    private static final int BEGIN_PORT = 8000;
    private static final AttributeKey<Object> SERVER_NAME_KEY = AttributeKey.newInstance("serverName");
    private static final String SERVER_NAME_VALUE = "nettyServer";
    public static final AttributeKey<Object> CLIENT_KEY = AttributeKey.newInstance("clientKey");
    public static final String CLIENT_VALUE = "clientValue";

    /**
     * 创建两个NioEventLoopGroup,这两个对象可以看做是传统IO编程模型的两大线程组,boosGroup表示监听端口,创建新连接的线程组,workerGroup表示处理每一条连接的数据读写的线程组
     * 创建引导类 ServerBootstrap进行服务端的启动工作,通过.group(boosGroup, workerGroup)给引导类配置两大线程定型引导类的线程模型指定服务端的IO模型为NIO,通过.channel(NioServerSocketChannel.class)来指定IO模型
     * 调用childHandler()方法给引导类创建ChannelInitializer定义后续每条连接的数据读写,业务处理逻辑,泛型参数NioSocketChannel是Netty对NIO类型的连接的抽象,而NioServerSocketChannel也是对NIO类型的连接的抽象
     * serverBootstrap.bind()是异步的方法调用之后是立即返回的,返回值是ChannelFuture,给ChannelFuture添加监听器GenericFutureListener,在GenericFutureListener的operationComplete方法里面监听端口是否绑定成功
     * childHandler()用于指定处理新连接数据的读写处理逻辑,handler()用于指定在服务端启动过程中的一些逻辑
     * attr()方法给服务端的channel即NioServerSocketChannel指定一些自定义属性,通过channel.attr()取出该属性,给NioServerSocketChannel维护一个map
     * childAttr()方法给每一条连接指定自定义属性,通过channel.attr()取出该属性
     * childOption()方法给每条连接设置一些TCP底层相关的属性:
     * ChannelOption.SO_KEEPALIVE表示是否开启TCP底层心跳机制,true为开启
     * ChannelOption.SO_REUSEADDR表示端口释放后立即就可以被再次使用,因为一般来说,一个端口释放后会等待两分钟之后才能再被使用
     * ChannelOption.TCP_NODELAY表示是否开始Nagle算法,true表示关闭,false表示开启,通俗地说,如果要求高实时性,有数据发送时就马上发送,就关闭,如果需要减少发送次数减少网络交互就开启
     * option()方法给服务端channel设置一些TCP底层相关的属性:
     * ChannelOption.SO_BACKLOG表示系统用于临时存放已完成三次握手的请求的队列的最大长度,如果连接建立频繁,服务器处理创建新连接较慢,适当调大该参数
     *
     * @param args
     */
    public static void main(String[] args) {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();

        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .handler(new ChannelInitializer<ServerSocketChannel>() {
                    @Override
                    protected void initChannel(ServerSocketChannel ch) throws Exception {
                        System.out.println("服务端启动中");
                        System.out.println(ch.attr(SERVER_NAME_KEY).get());
                    }
                })
                .attr(SERVER_NAME_KEY, SERVER_NAME_VALUE)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .childAttr(CLIENT_KEY, CLIENT_VALUE)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childOption(ChannelOption.SO_REUSEADDR, true)
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        System.out.println(ch.attr(CLIENT_KEY).get());
                    }
                });

        bind(serverBootstrap, BEGIN_PORT);
    }

    private static void bind(ServerBootstrap serverBootstrap, int port) {
        serverBootstrap.bind(BEGIN_PORT).addListener(new GenericFutureListener<Future<? super Void>>() {
            @Override
            public void operationComplete(Future<? super Void> future) throws Exception {
                if (future.isSuccess()) {
                    System.out.println("端口[" + port + "]绑定成功!");
                } else {
                    System.err.println("端口[" + port + "]绑定失败!");
                    bind(serverBootstrap, port + 1);
                }
            }
        });
    }
}
```

#### 2.客户端启动流程

要启动Netty客户端,必须要指定三类属性,分别是线程模型、IO模型、连接读写处理逻辑.
Netty客户端启动流程:创建引导类->指定线程模型、IO模型、连接读写处理逻辑->建立连接.

![](https://upload-images.jianshu.io/upload_images/8387919-a999eba6bf010539.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

失败重连通过connect()异步回调机制实现指数退避重连逻辑:

```java
// 第几次重连
int order = (MAX_RETRY - retry) + 1;
// 本次重连的间隔
int delay = 1 << order;
bootstrap.config().group().schedule(() -> connect(bootstrap, host, port, retry - 1), delay, TimeUnit.SECONDS);
```

### 三、客户端与服务端双向通信

- 客户端/服务端连接读写逻辑处理均是启动阶段通过给逻辑处理链 Pipeline 添加逻辑处理器实现连接数据的读写逻辑.
-  客户端连接成功回调逻辑处理器channelActive()方法,客户端/服务端接收连接数据调用channelRead()方法.
-  写数据调用writeAndFlush()方法,客户端与服务端交互的二进制数据传输载体为ByteBuf,ByteBuf通过连接的内存管理器创建即ctx.alloc().buffer(),通过writeBytes()方法将字节数据填充到ByteBuf 写到对端.

### 四、数据传输载体ByteBuf介绍

![](https://upload-images.jianshu.io/upload_images/8387919-3d38d8dc3509e5bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

二进制数据抽象ByteBuf是字节容器,容器里面的的数据分为三个部分,第一个部分是已经丢弃的字节,这部分数据是无效的;第二部分是可读字节,这部分数据是ByteBuf的主体数据,从ByteBuf里面读取的数据都来自这一部分;最后一部分的数据是可写字节,所有写到 ByteBuf的数据都会写到这一段.最后一部分虚线表示的是该ByteBuf最多还能扩容多少容量.这三部分内容是被两个指针给划分出来的,从左到右依次是读指针(readerIndex)、写指针(writerIndex),然后还有容量capacity表示ByteBuf底层内存的总容量.
 读数据是从ByteBuf里每读取一个字节,readerIndex自增1,ByteBuf里面共有writerIndex-readerIndex个字节可读, 由此推论当readerIndex与writerIndex相等的时候,ByteBuf不可读.
 写数据是从writerIndex指向的部分开始写,每写一个字节,writerIndex 自增1,直到增加至容量capacity,此时表示ByteBuf不可写.
 ByteBuf里容量上限maxCapacity表示当向ByteBuf写数据时容量capacity不足进行扩容,直至扩容到最大容量maxCapacity,超过 maxCapacity报错.
 推荐API:markReaderIndex()&resetReaderIndex()/markWriterIndex()& resetWriterIndex():前者表示把当前的读/写指针保存起来,后者表示把当前的读/写指针恢复到之前保存的值,常见使用场景为解析自定义协议的数据包.
 解析数据注意:get/set()方法不会改变读写指针,read/write()方法会改变读写指针.
 Netty使用堆外内存,堆外内存是不被JVM直接管理的,申请到的内存无法被垃圾回收器直接回收需要手动回收,即申请到的内存必须手工释放,否则造成内存泄漏.
 ByteBuf通过引用计数方式管理,如果ByteBuf没有地方被引用到,需要回收底层内存.默认情况下,当创建完ByteBuf其引用为1,然后每次调用retain()方法引用加1,release()方法原理是将引用计数减1,减完发现引用计数为0回收ByteBuf底层分配内存.
 slice()/duplicate()/copy():
 1.slice()方法从原始ByteBuf截取一段,这段数据是从readerIndex到writeIndex,返回的新的ByteBuf的最大容量maxCapacity为原始ByteBuf的readableBytes();
 2.duplicate()方法把整个ByteBuf都截取出来包括所有的数据、指针信息;
 3.slice()方法与duplicate()方法的相同点是:底层内存以及引用计数与原始的ByteBuf共享即经过slice()或者duplicate()返回的ByteBuf调用 write系列方法都会影响到原始的ByteBuf,但是它们都维持着与原始 ByteBuf相同的内存引用计数和不同的读写指针;
 4.slice()方法与duplicate()方法的不同点是:slice()只截取从readerIndex到writerIndex之间的数据,返回的ByteBuf最大容量被限制到原始 ByteBuf的readableBytes(), duplicate()是把整个ByteBuf都与原始的 ByteBuf共享;
 5.slice()方法与duplicate()方法不会拷贝数据,它们只是通过改变读写指针来改变读写的行为,copy()直接从原始的ByteBuf拷贝所有的信息包括读写指针以及底层对应的数据,因此往copy()返回的ByteBuf写数据不会影响到原始的ByteBuf;
 6.slice()方法与duplicate()方法不会改变ByteBuf的引用计数,所以原始的ByteBuf调用release()之后发现引用计数为零开始释放内存,调用这两个方法返回的ByteBuf也会被释放,此时如果再对它们进行读写报错.因此通过调用一次retain()方法增加引用,表示它们对应的底层的内存多一次引用,引用计数为2,在释放内存的时候需要调用两次 release()方法将引用计数降到零才会释放内存;
 7.slice()/duplicate()/copy()方法均维护着自身的读写指针,与原始的 ByteBuf的读写指针无关,相互之间不受影响.
 函数体里面只要增加引用计数包括ByteBuf的创建和手动调用 retain()方法必须调用release()方法.多个ByteBuf引用同一段内存通过引用计数来控制内存的释放,遵循谁retain()谁release()的原则.

### 五、客户端与服务端通信协议编解码

客户端与服务端的通信协议是客户端与服务端事先商量好的,每一个二进制数据包每一段字节分别代表什么含义的规则.

客户端与服务端的通信过程:首先客户端把Java对象按照通信协议转换成二进制数据包;然后通过网络把这段二进制数据包发送到服务端,数据的传输过程由TCP/IP协议负责数据的传输;服务端接收到数据按照协议截取二进制数据包的相应字段包装成Java对象交给应用逻辑处理;服务端处理完毕如果需要返回响应给客户端按照相同的流程进行.

![](https://upload-images.jianshu.io/upload_images/8387919-eaf1f38030bd881f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通信协议的设计:第一个字段是魔数,通常情况下为固定的几个字节;接下来一个字节为版本号,通常情况下是预留字段用于协议升级;第三部分序列化算法表示如何把 Java 对象转换二进制数据以及二进制数据如何转换回Java对象;第四部分的字段表示指令,服务端或者客户端每收到一种指令都有相应的处理逻辑;接下来的字段为数据部分的长度;最后一个部分为数据内容.

![](https://upload-images.jianshu.io/upload_images/8387919-9c13aca3f50d736a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

私有通信协议设计参考蚂蚁金服[SOFABolt](https://github.com/alipay/sofa-bolt)基于Netty实现的网络通信框架[蚂蚁通信框架实践](https://mp.weixin.qq.com/s/JRsbK1Un2av9GKmJ8DK7IQ)案例.

![](https://upload-images.jianshu.io/upload_images/8387919-d50e186de89f386e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 通信协议的实现:把Java对象根据协议封装成二进制数据包的过程称为编码;把从二进制数据包中解析出Java对象的过程称为解码.
- 编码(封装成二进制过程)流程:1.调用ByteBuf分配器ByteBufAllocator创建ByteBuf对象,ioBuffer()方法返回适配io读写相关的内存,尽可能创建直接内存,写到 IO 缓冲区的效果更高;2.把Java对象序列化成二进制数据包;3.按照通信协议逐个往ByteBuf对象写入字段即实现编码.
- 解码(解析Java对象过程)流程:1.调用skipBytes()方法跳过魔数 0x123
 45678字节;2.调用ByteBuf的API获取序列化算法标识、指令、数据包长度;3.根据获取的数据包长度取出数据,通过指令拿获取数据包对应的Java对象类型,根据序列化算法标识获取序列化对象,将字节数组转换为Java对象即实现解码.




### 参考资料
- [Netty入门与实战:仿写微信IM即时通讯系统](https://www.jianshu.com/p/7522bda72a25)
-  [闪电侠 Netty 小册里的骚操作](https://www.jianshu.com/p/bb0805d65388)
-  [思维导图](https://www.processon.com/view/link/5bc343dde4b09b21f31baf69)
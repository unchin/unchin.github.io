# 实时语音转写设计文档 - 用钦

## WebSocket 协议

  

### 1. 概述

  

相比 HTTP 协议来说，WebSocket 协议对大多数后端开发者是比较陌生的。相比来说，WebSocket 协议**重点**是提供了服务端主动向客户端发送数据的能力，这样我们就可以完成**实时性**较高的需求。例如说，聊天 IM 即使通讯功能、消息订阅服务、网页游戏等等。

  

同时，因为 WebSocket 使用 TCP 通信，可以避免重复创建连接，提升通信质量和效率。

### 2. 问题：WebSocket 为什么要依赖于 HTTP 协议的连接？

  

第一，WebSocket 设计上就是天生为HTTP增强通信，所以在 HTTP 协议连接的基础上是很自然的一件事。

  

第二，基于 HTTP 连接将获得最大的一个兼容支持，比如即使服务器不支持 WebSocket 也能建立 HTTP 通信，只不过返回的是 onerror 而已，这显然比服务器无响应要好的多。

### 3. 问题：**如何保证消息一定送达给用户**？

1. 定时轮询 ack 机制
2. 滑动窗口 ack 机制

### 4. demo 基于 Java

Java 实现 WebSocket 协议有多种方式，基于 Tomcat、Spring、Netty都行，目前业内最常用的就是 Netty，这里的 demo 仅为演示，基于 Tomcat 实现。

#### 4.1. 服务端

```
// WebsocketServerEndpoint.java

@Controller
@ServerEndpoint("/")
public class WebsocketServerEndpoint {

    private Logger logger = LoggerFactory.getLogger(getClass());

    @OnOpen
    public void onOpen(Session session, EndpointConfig config) {
        logger.info("[onOpen][session({}) 接入]", session);
    }

    @OnMessage
    public void onMessage(Session session, String message) {
        logger.info("[onOpen][session({}) 接收到一条消息({})]", session, message); // 生产环境下，请设置成 debug 级别
    }

    @OnClose
    public void onClose(Session session, CloseReason closeReason) {
        logger.info("[onClose][session({}) 连接关闭。关闭原因是({})}]", session, closeReason);
    }

    @OnError
    public void onError(Session session, Throwable throwable) {
        logger.info("[onClose][session({}) 发生异常]", session, throwable);
    }

}
```

- 在类上，添加 @Controller 注解，保证创建一个 WebsocketServerEndpoint Bean 。
- 在类上，添加 JSR-356 定义的 [@ServerEndpoint](https://github.com/eclipse-ee4j/websocket-api/blob/master/api/server/src/main/java/javax/websocket/server/ServerEndpoint.java) 注解，标记这是一个 WebSocket EndPoint ，路径为 / 。
- WebSocket 一共有四个事件，分别对应使用 JSR-356 定义的 [@OnOpen](https://github.com/eclipse-ee4j/websocket-api/blob/master/api/client/src/main/java/javax/websocket/OnOpen.java)、[@OnMessage](https://github.com/eclipse-ee4j/websocket-api/blob/master/api/client/src/main/java/javax/websocket/OnMessage.java)、[@OnClose](https://github.com/eclipse-ee4j/websocket-api/blob/master/api/client/src/main/java/javax/websocket/OnClose.java)、[@OnError](https://github.com/eclipse-ee4j/websocket-api/blob/master/api/client/src/main/java/javax/websocket/OnError.java) 注解。

#### 4.2. 客户端

```
/**
 * MyWebSocketClient
 *
 * @author guoyongqin
 * @version 2023/10/21 11:23
 **/
public class MyWebSocketClient extends WebSocketClient {

    private final CountDownLatch handshakeSuccess;

    private final CountDownLatch connectClose;

    public MyWebSocketClient(URI serverUri, Draft protocolDraft) {
        super(serverUri, protocolDraft);
        this.handshakeSuccess = handshakeSuccess;
        this.connectClose = connectClose;
    }

    @Override
    public void onOpen(ServerHandshake handshake) {
        LOGGER.info(getCurrentTimeStr() + "\t连接建立成功！");
    }

    @Override
    public void onMessage(String msg) {
        JSONObject msgObj = JSON.parseObject(msg);
        String action = msgObj.getString("action");
        if (Objects.equals("started", action)) {
            // 握手成功
            LOGGER.info(getCurrentTimeStr() + "\t握手成功！sid: " + msgObj.getString("sid"));
            handshakeSuccess.countDown();
        } else if (Objects.equals("result", action)) {
            // 转写结果
            String data = msgObj.getString("data");
            LOGGER.info(getCurrentTimeStr() + "\tresult: " + getContent(data));
        } else if (Objects.equals("error", action)) {
            // 连接发生错误
            LOGGER.info(getCurrentTimeStr() + "\tError: " + msg);
            connectClose.countDown();
        }
    }
    
    @Override
    public void onError(Exception e) {
        String currentTimeStr = getCurrentTimeStr();
        String errorMessage = e.getMessage();
        String logMessage = String.format("%s\t连接发生错误：%s, %s", currentTimeStr, errorMessage, new Date());
        LOGGER.error(logMessage);
    }

    @Override
    public void onClose(int arg0, String arg1, boolean arg2) {
        LOGGER.info(getCurrentTimeStr() + "\t链接关闭");
        connectClose.countDown();
    }
}
```

  

### 5. 参考文献

1. 芋道源码：[https://www.iocoder.cn/Spring-Boot/WebSocket/](https://www.iocoder.cn/Spring-Boot/WebSocket/)
2. 阮一峰的网络日志：[https://www.ruanyifeng.com/blog/2017/05/websocket.html](https://www.ruanyifeng.com/blog/2017/05/websocket.html)

## Netty 框架

### 1. 为什么要使用 Netty

1. 使用 JDK 自带的NIO需要了解太多的概念，编程复杂，一不小心 bug 横飞
2. Netty 底层 IO 模型随意切换，而这一切只需要做微小的改动，改改参数，Netty可以直接从 NIO 模型变身为 IO 模型
3. Netty 自带的拆包解包，异常检测等机制让你从NIO的繁重细节中脱离出来，让你只需要关心业务逻辑
4. Netty 解决了 JDK 的很多包括空轮询在内的 Bug
5. Netty 底层对线程，selector 做了很多细小的优化，精心设计的 reactor 线程模型做到非常高效的并发处理
6. 自带各种协议栈让你处理任何一种通用协议都几乎不用亲自动手
7. Netty 社区活跃，遇到问题随时邮件列表或者 issue
8. Netty 已经历各大 RPC 框架，消息中间件，分布式通信中间件线上的广泛验证，健壮性无比强大

### 2. ByteBuf 数据传输载体

Netty 里面数据读写是以 ByteBuf 为单位进行交互，下面是它的结构

![](https://cdn.nlark.com/yuque/0/2023/png/432307/1702435722347-1f84185b-26b9-40ed-88ba-459f590e25cb.png)

从这幅图可以看到

1. ByteBuf 是一个字节容器，容器里面的的数据分为三个部分，第一个部分是已经丢弃的字节，这部分数据是无效的；第二部分是可读字节，这部分数据是 ByteBuf 的主体数据， 从 ByteBuf 里面读取的数据都来自这一部分;最后一部分的数据是可写字节，所有写到 ByteBuf 的数据都会写到这一段。最后一部分虚线表示的是该 ByteBuf 最多还能扩容多少容量
2. 以上三段内容是被两个指针给划分出来的，从左到右，依次是读指针（readerIndex）、写指针（writerIndex），然后还有一个变量 capacity，表示 ByteBuf 底层内存的总容量
3. 从 ByteBuf 中每读取一个字节，readerIndex 自增1，ByteBuf 里面总共有 writerIndex-readerIndex 个字节可读, 由此可以推论出当 readerIndex 与 writerIndex 相等的时候，ByteBuf 不可读
4. 写数据是从 writerIndex 指向的部分开始写，每写一个字节，writerIndex 自增1，直到增到 capacity，这个时候，表示 ByteBuf 已经不可写了
5. ByteBuf 里面其实还有一个参数 maxCapacity，当向 ByteBuf 写数据的时候，如果容量不足，那么这个时候可以进行扩容，直到 capacity 扩容到 maxCapacity，超过 maxCapacity 就会报错

Netty 使用 ByteBuf 这个数据结构可以有效地区分可读数据和可写数据，读写之间相互没有冲突，当然，ByteBuf 只是对二进制数据的抽象，Netty 关于数据读写只认 ByteBuf。

### 3. Netty demo

#### 3.1. 服务端

```
public class NettyServer {
    public static void main(String[] args) {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap
                .group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    protected void initChannel(NioSocketChannel ch) {
                    }
                });
        serverBootstrap.bind(8000);
    }
}
```

- 首先看到，我们创建了两个 NioEventLoopGroup，这两个对象可以看做是传统IO编程模型的两大线程组，bossGroup 表示监听端口，accept 新连接的线程组，workerGroup 表示处理每一条连接的数据读写的线程组。用生活中的例子来讲就是，一个工厂要运作，必然要有一个老板负责从外面接活，然后有很多员工，负责具体干活，老板就是bossGroup，员工们就是workerGroup，bossGroup接收完连接，扔给workerGroup去处理。
- 接下来 我们创建了一个引导类 ServerBootstrap，这个类将引导我们进行服务端的启动工作，直接new出来开搞。
- 我们通过 .group(bossGroup, workerGroup) 给引导类配置两大线程组，这个引导类的线程模型也就定型了。
- 然后，我们指定我们服务端的 IO 模型为 NIO，我们通过 .channel(NioServerSocketChannel.class) 来指定 IO 模型，当然，这里也有其他的选择，如果你想指定 IO 模型为 BIO，那么这里配置上OioServerSocketChannel.class 类型即可，当然通常我们也不会这么做，因为 Netty 的优势就在于 NIO。
- 接着，我们调用 childHandler() 方法，给这个引导类创建一个 ChannelInitializer，这里主要就是定义后续每条连接的数据读写，业务处理逻辑。ChannelInitializer这个类中，我们注意到有一个泛型参数 NioSocketChannel，这个类呢，就是 Netty 对 NIO 类型的连接的抽象，而我们前面 NioServerSocketChannel也是对 NIO 类型的连接的抽象，NioServerSocketChannel 和 NioSocketChannel 的概念可以和 BIO 编程模型中的 ServerSocket 以及 Socket 两个概念对应上

我们的最小化参数配置到这里就完成了，我们总结一下就是，要启动一个Netty服务端，必须要指定三类属性，分别是线程模型、IO 模型、连接读写处理逻辑，有了这三者，之后在调用bind(8000)，我们就可以在本地绑定一个 8000 端口启动起来，以上这段代码可以直接拷贝到 IDE 中运行。

#### 3.2. 客户端

```
public class NettyClient {
    public static void main(String[] args) throws InterruptedException {
        Bootstrap bootstrap = new Bootstrap();//服务端是ServerBootstrap,客户端时Bootstrap
        NioEventLoopGroup group = new NioEventLoopGroup();
        bootstrap
                .group(group)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel ch) {
                        ch.pipeline().addLast(new StringEncoder());
                    }
                });
        Channel channel = bootstrap.connect("127.0.0.1", 8000).addListener(future -> {
            if (future.isSuccess()) {
                System.out.println("连接成功!");
            } else {
                System.err.println("连接失败!");
            }
        }).channel();
        while (true) {
            channel.writeAndFlush(new Date() + ": hello world!");
            Thread.sleep(2000);
        }
    }
}
```

对于客户端的启动来说，和服务端的启动类似，依然需要线程模型、IO 模型，以及 IO 业务处理逻辑三大参数。

### 4. 参考文献

1. [https://hogwartsrico.github.io/2020/07/15/Netty/index.html](https://hogwartsrico.github.io/2020/07/15/Netty/index.html)
2. [Netty.docs: New and noteworthy in 4.1](https://netty.io/wiki/new-and-noteworthy-in-4.1.html)
3. [小闪对话：微信聊天长连设计的探讨（一）](https://mp.weixin.qq.com/s?__biz=MzI1OTUzMTQyMA==&mid=2247484094&idx=1&sn=d3c89ca9897f11e94deaa85e16e09e8c&chksm=ea76354ddd01bc5b49da25fc47237137796e1151e69ad975d47d37241cfefcca3762ee35017e&token=1671319965&lang=zh_CN#rd)

## 实战

### 1. Netty 服务端

#### 1.1. 启动类 NettyChatServer

首先创建启动类，将初始化的方法封装为类，添加到启动线程组中，Netty 中的线程数默认是 CPU 处理器数的两倍，bind 完之后启动。

```
ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.SO_KEEPALIVE, true).childOption(ChannelOption.TCP_NODELAY, true)
                .childHandler(nettyChatServerInitializer);
            bind(serverBootstrap, serverNettyPort);
```

这里的 bind 方法，我们可以使用递归调用的方式实现多次重连，毕竟有时候会因为服务器异常导致端口无法正常使用。

```
private void bind(final ServerBootstrap serverBootstrap, final int port) throws InterruptedException {
        ChannelFuture channelFuture = serverBootstrap.bind(port).addListener(future -> {
            if (future.isSuccess()) {
                log.info(new Date() + ": 端口[" + port + "]绑定成功!");
            } else {
                log.info("端口[" + port + "]绑定失败!");
                bind(serverBootstrap, port + 1);
            }
        });
        channelFuture.channel().closeFuture().sync();
    }
```

#### 1.2. 初始化类 NettyChatServerInitializer

在初始化类中就是 Netty 使用的责任链模式的体现，底层采用双向链表的数据结构, 将链上的各个处理器串联起来。

客户端每一个请求的到来,Netty 都认为,pipeline 中的所有的处理器都有机会处理它,因此,对于入栈的请求,全部从头节点开始往后传播,一直传播到尾节点(来到尾节点的msg会被释放掉)。

```
/**
 * netty 服务端初始化类
 *
 * @author guoyongqin
 * @version 2023/10/16 17:27
 **/
@RequiredArgsConstructor
@Component
public class NettyChatServerInitializer extends ChannelInitializer<SocketChannel> {

    private  final NettyChatServerHandler nettyChatServerHandler;
    @Override
    protected void initChannel(SocketChannel ch){
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new HttpServerCodec());
        pipeline.addLast(new ChunkedWriteHandler());
        pipeline.addLast(new HttpObjectAggregator(65536));
        pipeline.addLast(new WebSocketServerCompressionHandler());
        String path = "/";
        pipeline.addLast(new WebSocketServerProtocolHandler(path, null, true, Math.toIntExact(1048576)));
        pipeline.addLast(nettyChatServerHandler);
    }
}
```

我们在这个管道中陆续添加了编解码器、大数据流量控制器、Http 消息聚合处理器、消息压缩和解压缩处理器和WebSocket 握手和协议升级处理器，最后加上我们自己的自定义处理器。

#### 1.3. 自定义消息处理器 NettyChatServerHandler

```
@ChannelHandler.Sharable
@Component
@RequiredArgsConstructor
public class NettyChatServerHandler extends SimpleChannelInboundHandler<TextWebSocketFrame>{

    /**
     * 1. 解析拿到的 json 为 BasePacket 2. 通过 BasePacket 中的指令编码拿到 自定义的消息对象 和 响应的单例
     */
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) {
        try {
            String text = Optional.ofNullable(msg.text()).orElseThrow(() -> new NotFoundException("消息不存在"));
            BasePacket basePacket = getBasePacket(ctx, text);

            Byte command = Optional.of(basePacket.getCommand()).orElseThrow(() -> new NotFoundException("指令不存在"));
            BasePacket customPacket = getCustomPacket(basePacket, command);
            
            RequestHandler requestHandler = serverHandlerFactory.getHandler(command);
            requestHandler.handler(ctx, customPacket);
        } catch (Exception e) {
            ctx.writeAndFlush(new TextWebSocketFrame("ws连接异常：" + e.getMessage()));
            LOGGER.error("ws连接异常：", e);
        }
    }

    @NotNull
    private BasePacket getCustomPacket(BasePacket basePacket, Byte command) throws NotFoundException {
        String content = Optional.of(basePacket.getContent()).orElseThrow(() -> new NotFoundException("消息内容不存在"));
        JSONObject jsonObject = JSON.parseObject(content);
        Class<? extends BasePacket> aClass = packetMap.get(command);
        BasePacket packet = jsonObject.toJavaObject(aClass);
        packet.setToken(basePacket.getToken());
        return packet;
    }

    @NotNull
    private BasePacket getBasePacket(ChannelHandlerContext ctx, String text) throws NotFoundException {
        JSONObject textJson = JSON.parseObject(text);
        BasePacket basePacket = textJson.toJavaObject(BasePacket.class);
        return basePacket;
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        ctx.close();
    }

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) {}

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) {}

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        // 连接成功的操作
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        // 断开连接的操作
    }
}
```

![](https://cdn.nlark.com/yuque/0/2023/png/432307/1703575689969-cbb81b2c-5554-48ff-bc7d-473179aa8683.png)

### 2. 创建会议室

```
/**
 * 创建群聊请求消息处理器
 *
 * @author guoyongqin
 */
@Component
@RequiredArgsConstructor
public class MeetInitRequestHandler extends AbstractRequestHandler
    implements MeetRequestHandler<MeetInitRequestPacket> {
    @Override
    public void handler(ChannelHandlerContext ctx, MeetInitRequestPacket meetInitRequestPacket) throws Exception {
        String groupId = meetInitRequestPacket.getGroupId();
        MeetInitResponsePacket r = getCreateGroupResponsePacket(meetInitRequestPacket);

        // 加入群聊
        joinGroup(ctx, meetInitRequestPacket, groupId);
        ctx.writeAndFlush(new TextWebSocketFrame(JSONObject.toJSONString(r)));
    }
```

### 3. 会议室广播消息（群聊）

```
public class MeetMessageRequestHandler extends AbstractRequestHandler
    implements MeetRequestHandler<MeetMessageRequestPacket> {
    @Override
    public void handler(ChannelHandlerContext ctx, MeetMessageRequestPacket packet) throws Exception {
        String groupId = packet.getToGroupId();
        // 发送到ai转写服务
        aiSendTemplate.send(groupId, packet.getCommand(), packet.getMessage());
        super.addChannelGroup(ctx, groupId);
    }
}
```

### 4. 加入会议

```
public class MeetJoinRequestHandler extends AbstractRequestHandler
    implements MeetRequestHandler<MeetJoinRequestPacket> {

    @Override
    public void handler(ChannelHandlerContext ctx, MeetJoinRequestPacket packet) throws Exception {
        String groupId = packet.getGroupId();

        // 1.查询会议以及参会人校验
        	...	
        // 2. 构造加群响应发送给客户端
        responseResult(ctx, packet, groupId, meet, participants);
        super.addChannelGroup(ctx, groupId);
    }

    private void responseResult(ChannelHandlerContext ctx, MeetJoinRequestPacket packet, String groupId, Meet meet,
        List<Participants> participants) {
        MeetJoinResponsePacket responsePacket = new MeetJoinResponsePacket();
        ...
        responsePacket.setContent(packet.isReturnHistory() == true ? getHistoryMessage(groupId) : null);
        ctx.writeAndFlush(new TextWebSocketFrame(JSONObject.toJSONString(responsePacket)));
    }

    private String getHistoryMessage(String groupId) {
        // 从mongo里面获取历史消息
        List<RtAsrMessageVo> list = getRtAsrMessageVos(groupId);
        if (CollectionUtils.isEmpty(list)) {
            return StringUtils.EMPTY;
        }
        // 将片段信息汇总成整个片段给到前端
        return buildHistoryMessage(list);
    }
}
```

### 5. 与 springboot 中 tomcat 进程保持生命周期一致

```
public class NettyChatServer implements ApplicationRunner {
```

```
public class ServerHandlerFactory implements ApplicationContextAware {

    @Getter
    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext){
        this.applicationContext = applicationContext;
    }

    // 使用
    public static RequestHandler getHandler(Byte command) {
        return applicationContext.getBean(getClazz(command));
    }
}
```

### 6. 链路打通

![](https://cdn.nlark.com/yuque/0/2023/png/432307/1703296918922-8ea14824-48d1-43b0-b575-d4bfbda17351.png)

1. ai 服务启动 Netty 服务端，并与 springboot 生命周期保持同步

```
public class AiServer implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            log.info("ai-websocket 开始启动");
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.SO_KEEPALIVE, true).childOption(ChannelOption.TCP_NODELAY, true)
                .childHandler(serverInitializer);
            bind(serverBootstrap, port);
        } catch (Exception e) {
            log.error("ai-websocket 服务启动失败", e);
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
```

2. meet 服务启动 Netty 客户端连接 ai 服务，并与 springboot 生命周期保持同步

```
public class AiClient implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        connect();
    }

    public void connect() throws NacosException {
        List<Instance> aiInstances = getAiInstances();
        if (!CollectionUtils.isEmpty(aiInstances)) {
            for (Instance instance : aiInstances) {
                String host = instance.getIp();
                Channel channel = aiClientContext.getClientChannel(aiClientContext.sessionKey(host, port));
                if (Objects.nonNull(channel) && channel.isActive()) {
                    continue;
                }
                connect(host);
            }
        }
    }

    private void connect(String host) {
        Bootstrap bootstrap = new Bootstrap();
        NioEventLoopGroup workGroup = new NioEventLoopGroup();
        bootstrap.group(workGroup).channel(NioSocketChannel.class).option(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.TCP_NODELAY, true).handler(aiClientInitializer);

        bootstrap.connect(host, port).addListener(future -> {
            if (future.isSuccess()) {
                log.info("连接AI服务成功, host:{}, port:{}", host, port);
                Channel channel = ((ChannelFuture)future).channel();
                AiClientSession aiClientSession = new AiClientSession(host, port);
                channel.attr(AiClientAttributes.CLIENT_SESSION).set(aiClientSession);
                aiClientContext.addClientChannel(aiClientContext.sessionKey(host, port), channel);
            }
        });
    }
}
```

### 7. 多实例

![](https://cdn.nlark.com/yuque/0/2023/png/432307/1703229923470-67d127db-f43e-4d9d-a7ef-579f9df612bf.png)

  

1. meet 服务校验群组

```
protected ChannelGroup addChannelGroup(ChannelHandlerContext ctx, String groupId) {
        ChannelGroup channelGroup = SessionUtil.getChannelGroup(groupId);
        if (Objects.isNull(channelGroup)) {
            channelGroup = new DefaultChannelGroup(ctx.executor());
            SessionUtil.bindChannelGroup(groupId, channelGroup);
        }
        if (!channelGroup.contains(ctx.channel())) {
            channelGroup.add(ctx.channel());
        }

        return channelGroup;
    }
```

2. meet 服务往 ai 服务发送消息校验客户端通道

```
public static void send(String groupId, AiPacket aiPacket) {
        Channel chanel = (Channel)groupChannelMap.get(groupId);
        ChannelFuture cf = null;
        if (Objects.nonNull(chanel) && chanel.isActive()) {
            cf = chanel.writeAndFlush(aiPacket);
        } else {
            Iterator var4 = clientChannelMap.entrySet().iterator();

            while(var4.hasNext()) {
                Map.Entry<String, Channel> entry = (Map.Entry)var4.next();
                Channel channel = (Channel)entry.getValue();
                if (channel.isActive()) {
                    groupChannelMap.put(groupId, channel);
                    channelGroupMap.put(channel, groupId);
                    cf = channel.writeAndFlush(aiPacket);
                    break;
                }
            }
        }

        if (Objects.isNull(cf)) {
            log.warn("发送失败:" + JSON.toJSONString(aiPacket));
        } else {
            cf.addListener((future) -> {
                if (future.isSuccess()) {
                    log.info("群组:{}发送成功:", groupId);
                } else {
                    log.warn("群组:{}发送失败:", groupId);
                    remove(chanel);
                }

            });
        }
    }
```

## 遇到的问题

### 1. 粘包

TCP是以流的方式来处理数据，一个完整的包可能会被TCP拆分成多个包进行发送，也可能把小的封装成一个大的数据包发送。应用程序写入的字节大小大于套接字发送缓冲区的大小，会发生拆包现象，而应用程序写入数据小于套接字缓冲区大小，网卡将应用多次写入的数据发送到网络上，这将会发生粘包现象；

拆包和粘包是相对的，一端粘了包，另外一端就需要将粘过的包拆开，举个栗子，发送端将三个数据包粘成两个 TCP 数据包发送到接收端，接收端就需要根据应用协议将两个数据包重新组装成三个数据包。

#### Netty 自带的拆包器

1. 固定长度的拆包器 FixedLengthFrameDecoder  
    如果你的应用层协议非常简单，每个数据包的长度都是固定的，比如 100，那么只需要把这个拆包器加到 pipeline 中，Netty 会把一个个长度为 100 的数据包 (ByteBuf) 传递到下一个 channelHandler。
2. 行拆包器 LineBasedFrameDecoder  
    从字面意思来看，发送端发送数据包的时候，每个数据包之间以换行符作为分隔，接收端通过 LineBasedFrameDecoder 将粘过的 ByteBuf 拆分成一个个完整的应用层数据包。
3. 分隔符拆包器 DelimiterBasedFrameDecoder  
    DelimiterBasedFrameDecoder 是行拆包器的通用版本，只不过我们可以自定义分隔符。
4. 基于长度域拆包器 LengthFieldBasedFrameDecoder  
    最通用的一种拆包器，只要你的自定义协议中包含长度域字段，均可以使用这个拆包器来实现应用层拆包。

Generally frame detection should be handled earlier in the pipeline by adding a DelimiterBasedFrameDecoder, FixedLengthFrameDecoder, LengthFieldBasedFrameDecoder, or LineBasedFrameDecoder.

```
    public LengthFieldBasedFrameDecoder(
            int maxFrameLength,
            int lengthFieldOffset, int lengthFieldLength) {
        this(maxFrameLength, lengthFieldOffset, lengthFieldLength, 0, 0);
    }
```

#### 自定义通信协议的设计

![](https://cdn.nlark.com/yuque/0/2023/png/432307/1703208453495-76fb5bd1-4099-484e-9612-fda2ab9464ad.png)

首先，第一个字段是魔数，通常情况下为固定的几个字节（我们这边规定为4个字节）。 为什么需要这个字段，而且还是一个固定的数？假设我们在服务器上开了一个端口，比如 80 端口，如果没有这个魔数，任何数据包传递到服务器，服务器都会根据自定义协议来进行处理，包括不符合自定义协议规范的数据包。例如，我们直接通过 [http://服务器ip](http://xn--ip-fr5c86lx7z) 来访问服务器（默认为 80 端口）， 服务端收到的是一个标准的 HTTP 协议数据包，但是它仍然会按照事先约定好的协议来处理 HTTP 协议，显然，这是会解析出错的。而有了这个魔数之后，服务端首先取出前面四个字节进行比对，能够在第一时间识别出这个数据包并非是遵循自定义协议的，也就是无效数据包，为了安全考虑可以直接关闭连接以节省资源。在 Java 的字节码的二进制文件中，开头的 4 个字节为0xcafebabe 用来标识这是个字节码文件，亦是异曲同工之妙。接下来一个字节为版本号，通常情况下是预留字段，用于协议升级的时候用到，有点类似 TCP 协议中的一个字段标识是 IPV4 协议还是 IPV6 协议，大多数情况下，这个字段是用不到的，不过为了协议能够支持升级，我们还是先留着。  
第三部分，序列化算法表示如何把 Java 对象转换二进制数据以及二进制数据如何转换回 Java 对象，比如 Java 自带的序列化，json，hessian 等序列化方式。  
第四部分的字段表示指令，服务端或者客户端每收到一种指令都会有相应的处理逻辑，这里，我们用一个字节来表示，最高支持256种指令，对于我们这个 IM 系统来说已经完全足够了。  
接下来的字段为数据部分的长度，占四个字节。  
最后一个部分为数据内容，每一种指令对应的数据是不一样的，比如登录的时候需要用户名密码，收消息的时候需要用户标识和具体消息内容等等。

  

#### 通信协议的实现

我们把 Java 对象根据协议封装成二进制数据包的过程成为编码，而把从二进制数据包中解析出 Java 对象的过程成为解码。

```
@Data
@NoArgsConstructor
public class AiPacket {
    @JSONField(deserialize = false, serialize = false)
    private Byte version = 1;
    private Byte command;
    private String businessId;
    private String message;
}
```

以上是通信过程中 Java 对象的抽象类，可以看到，我们定义了一个版本号（默认值为 1 ）以及一个获取指令的抽象方法，所有的指令数据包都必须实现这个方法，这样我们就可以知道某种指令的含义。

Java 对象定义完成之后，接下来我们就需要定义一种规则，如何把一个 Java 对象转换成二进制数据，这个规则叫做 Java 对象的序列化。

```
public interface Serializer {

    Serializer DEFAULT = new JSONSerializer();

    /**
     * 序列化算法
     */
    byte getSerializerAlgorithm();

    /**
     * java 对象转换成二进制
     */
    byte[] serialize(Object object);

    /**
     * 二进制转换成 java 对象
     */
    <T> T deserialize(Class<T> clazz, byte[] bytes);
}
```

我们使用最简单的 json 序列化方式，使用阿里巴巴的 fastjson 作为序列化框架。

  

接下来把这部分的数据编码到通信协议的二进制数据包中去。

```
public void encode(ByteBuf byteBuf, AiPacket packet) {
        // 1. 序列化 java 对象
        byte[] bytes = Serializer.DEFAULT.serialize(packet);

        // 2. 实际编码过程
        byteBuf.writeInt(MAGIC_NUMBER);
        byteBuf.writeByte(packet.getVersion());
        byteBuf.writeByte(Serializer.DEFAULT.getSerializerAlgorithm());
        byteBuf.writeByte(packet.getCommand());
        byteBuf.writeInt(bytes.length);
        byteBuf.writeBytes(bytes);
    }
```

编码过程分为三个过程：首先，我们需要创建一个 ByteBuf，这里我们调用 Netty 的 ByteBuf 分配器来创建，ioBuffer() 方法会返回适配 io 读写相关的内存，它会尽可能创建一个直接内存，直接内存可以理解为不受 jvm 堆管理的内存空间，写到 IO 缓冲区的效果更高。  
接下来，我们将 Java 对象序列化成二进制数据包。  
一端实现了编码之后，Netty 会将此 ByteBuf 写到另外一端，另外一端拿到的也是一个 ByteBuf 对象，基于这个 ByteBuf 对象，就可以反解出在对端创建的 Java 对象，这个过程我们称作为解码。

```
public AiPacket decode(ByteBuf byteBuf) {
        // 跳过 magic number
        byteBuf.skipBytes(4);

        // 跳过版本号
        byteBuf.skipBytes(1);

        // 序列化算法
        byte serializeAlgorithm = byteBuf.readByte();

        // 指令
        byte command = byteBuf.readByte();

        // 数据包长度
        int length = byteBuf.readInt();

        byte[] bytes = new byte[length];
        byteBuf.readBytes(bytes);

        Serializer serializer = getSerializer(serializeAlgorithm);

        return serializer.deserialize(AiPacket.class, bytes);
    }
```

我们假定 decode 方法传递进来的 ByteBuf 已经是合法的，即首四个字节是我们前面定义的魔数 0x12345678，这里我们调用 skipBytes 跳过这四个字节。  
这里，我们暂时不关注协议版本，通常我们在没有遇到协议升级的时候，这个字段暂时不处理，因为，你会发现，绝大多数情况下，这个字段几乎用不着，但我们仍然需要暂时留着。  
接下来，我们调用 ByteBuf 的 API 分别拿到序列化算法标识、指令、数据包的长度。  
最后，我们根据拿到的数据包的长度取出数据，通过指令拿到该数据包对应的 Java 对象的类型，根据序列化算法标识拿到序列化对象，将字节数组转换为 Java 对象，至此，解码过程结束。

### 2. 动态ip的获取

在 meet 中要获取到 ai 服务的ip信息，需要引入 nacos 相关的配置项

```
    public void connect() throws NacosException {
        List<Instance> aiInstances = getAiInstances();
        if (!CollectionUtils.isEmpty(aiInstances)) {
            for (Instance instance : aiInstances) {
                String host = instance.getIp();
                Channel channel = aiClientContext.getClientChannel(aiClientContext.sessionKey(host, port));
                if (Objects.nonNull(channel) && channel.isActive()) {
                    continue;
                }
                connect(host);
            }
        }
    }
```

```
    public List<Instance> getAiInstances() throws NacosException {
        NamingService namingService = nacosServiceManager.getNamingService();
        return namingService.getAllInstances(instanceName, nacosDiscoveryProperties.getGroup());
    }
```

### 3. mongo查看历史聊天记录

加入会议者，也就是参会人，在会议进行的时候需要拿到历史的消息记录，之前的翻译记录我们都存在mongo中，所以在拿历史消息记录时，就需要进行 mongodb 的查询

```
    public List<RtAsrMessageVo> queryAsrMessageByGroupId(String groupId) {
        List<RtAsrMessageVo> result = Lists.newArrayList();
        // 取出会议室里最终结果(type=0)的数据条，按创建时间正序
        Criteria regex = Criteria.where("group_id").is(groupId).and("message").regex(".*\"type\":\"0\".*");
        Query query = new Query();
        query.addCriteria(regex);
        query.with(Sort.by("create_time"));
        List<RtAsrMessage> list = super.find(query);
        list = Optional.ofNullable(list).orElse(Lists.newArrayList());

        // 将list中的对象转成vo
    	...
        return result;
    }
```

### 4. 角色分离

很多时候，三方转写返回的数据格式，我们都需要重新处理成业务所需要的格式，而我们的实时转写需要有不同角色分离为不同的段落，于是我们就需要将返回的数据进行角色分离。

```
{
    "seg_id": 0,
    "cn": {
        "st": {
            "rt": [
                {
                    "ws": [
                        {
                            "cw": [
                                {
                                    "sc": 0,
                                    "w": "对",
                                    "wp": "n",
                                    "rl": "1",
                                    "wb": 1,
                                    "wc": 0,
                                    "we": 5
                                }
                            ],
                            "wb": 1,
                            "we": 5
                        },
                        {
                            "cw": [
                                {
                                    "sc": 0,
                                    "w": "返回",
                                    "wp": "n",
                                    "rl": "0",
                                    "wb": 5,
                                    "wc": 0,
                                    "we": 48
                                }
                            ],
                            "wb": 5,
                            "we": 48
                        },
                        {
                            "cw": [
                                {
                                    "sc": 0,
                                    "w": "的",
                                    "wp": "n",
                                    "rl": "0",
                                    "wb": 49,
                                    "wc": 0,
                                    "we": 72
                                }
                            ],
                            "wb": 49,
                            "we": 72
                        },
                        {
                            "cw": [
                                {
                                    "sc": 0,
                                    "w": "情",
                                    "wp": "n",
                                    "rl": "0",
                                    "wb": 73,
                                    "wc": 0,
                                    "we": 92
                                }
                            ],
                            "wb": 73,
                            "we": 92
                        },
                        {
                            "cw": [
                                {
                                    "sc": 0,
                                    "w": "况",
                                    "wp": "n",
                                    "rl": "0",
                                    "wb": 92,
                                    "wc": 0,
                                    "we": 112
                                }
                            ],
                            "wb": 92,
                            "we": 112
                        },
                        {
                            "cw": [
                                {
                                    "sc": 0,
                                    "w": "。",
                                    "wp": "p",
                                    "rl": "0",
                                    "wb": 112,
                                    "wc": 0,
                                    "we": 142
                                }
                            ],
                            "wb": 142,
                            "we": 142
                        }
                    ]
                }
            ],
            "bg": "0",
            "type": "0",
            "ed": "1010"
        }
    },
    "ls": true
}
```

这个数据中每一个词的"rl"属性就是不用的角色 id，通过这个属性进行角色分离

```
   /**
     * 将 List<RtAsrResultFrom> 中的 rl（说话人角色）字段相同 的src拼在一起
     *
     * @param result 转写的返回结果
     * @return List<RtAsrResultFrom> 解析后的结果列表
     */
    public static List<RtAsrResultFrom> assemblyMessage(List<RtAsrResultFrom> result) {
        List<RtAsrResultFrom> resultList = new ArrayList<>();
        StringBuilder src = new StringBuilder();
        int firstRl = -1;
        int wb = 0;
        int we = 0;
        int en = 0;
        int bg = 0;
        long segId = -1;

        // 获取result中的第一个对象
        if (!result.isEmpty()) {
            RtAsrResultFrom firstResult = result.get(0);
            firstRl = firstResult.getRl();
        }

        // 遍历列表中的所有对象，如果是rl一致的，将他们的内容拼接起来，如果不一致，就意味着断句，则将之前的内容添加到resultList中
        for (RtAsrResultFrom resultFrom : result) {
            if (Objects.equals(resultFrom.getRl(), firstRl)) {
                src.append(resultFrom.getSrc());
                bg += Math.toIntExact(resultFrom.getBg());
                en += resultFrom.getEn();
                segId = resultFrom.getSegId();
            } else {
                RtAsrResultFrom resultFrom1 = RtAsrResultFrom.builder()
                        .src(src.toString())
                        .rl(firstRl)
                        .wb(wb).we(we)
                        .bg((long) bg)
                        .en((long) en)
                        .segId(segId)
                        .build();
                resultList.add(resultFrom1);

                // 新的一句话，重置rl，src
                firstRl = resultFrom.getRl();
                src = new StringBuilder();
                src.append(resultFrom.getSrc());
            }
        }
        // 拼接的最后一句话也要添加到resultList中
        RtAsrResultFrom resultFrom2 = RtAsrResultFrom.builder()
                .src(src.toString())
                .rl(firstRl)
                .wb(wb)
                .we(we)
                .bg((long) bg)
                .en((long) en)
                .segId(segId)
                .build();
        resultList.add(resultFrom2);

        return resultList;
    }
```

解析结果

```
{
    "command": 10,
    "content": "[{\"bg\":452410,\"en\":454830,\"segId\":386,\"src\":\"，对\",\"type\":0},{\"bg\":452410,\"en\":454830,\"segId\":386,\"src\":\"，返回的情况。\",\"type\":0}]",
    "returnHistory": true,
    "success": true
}
```

### 5. 转写准确率

![](https://cdn.nlark.com/yuque/0/2023/png/432307/1703314301938-505ed6e3-a356-48c0-9cc0-8387305a0be8.png)

1. 尝试拼接前端传来的数据

1. 延迟高
2. 并发问题
3. 无法满足上下文
4. 角色无法正确分离

2. 保持ws连接，不用频繁new客户端

1. 流式输入如果流式输出，就无法关闭通道

![](https://cdn.nlark.com/yuque/0/2023/png/432307/1703314192110-1fd1b7d5-e962-48ce-9834-3aaaee8fe394.png)

```
    public void transcription(AiPacket aiPacket, RealTimeFunction realTimeFunction) throws Exception {
        CountDownLatch handshakeSuccess = new CountDownLatch(1);
        CountDownLatch connectClose = new CountDownLatch(1);

        String groupId = aiPacket.getBusinessId();
        // 根据会议号查询是否有 开启的转写的连接
        SocketClient existsClient = SocketContext.get(groupId);

        // 如果存在开启的转写的连接，则直接发送音频数据
        if (Objects.nonNull(existsClient) && existsClient.getReadyState().equals(WebSocket.READYSTATE.OPEN)) {
            sendMessage(existsClient, getBytes(aiPacket));
        }
        if (Objects.isNull(existsClient) || !existsClient.getReadyState().equals(WebSocket.READYSTATE.OPEN)) {
            // 如果不存在开启的转写的连接，则开启一个新的连接

            URI url = new URI(BASE_URL + getHandShakeParams(appId, secretKey));
            DraftWithOrigin draft = new DraftWithOrigin(ORIGIN);

            SocketClient client =
                new SocketClient(url, draft, handshakeSuccess, connectClose, groupId, realTimeFunction);
            client.connect();

            if (!client.getReadyState().equals(WebSocket.READYSTATE.OPEN)) {
                LOGGER.info(SDF.format(new Date()) + "\t连接中");
                Thread.sleep(1000);
            }

            // 等待握手成功
            handshakeSuccess.await();
            LOGGER.info(SDF.format(new Date()) + " 开始发送音频数据");
            // 发送音频
            sendMessage(client, getBytes(aiPacket));
        }

    }
```

```
public class AiServerQuitAsrPacketServiceImpl implements AiServerPacketService {

    private final AiServiceContext aiServiceContext;

    @Override
    public void handler(ChannelHandlerContext ctx, AiPacket aiPacket) throws Exception {

        // 关闭转写通道
        closeAsr(aiPacket);

        // 发送结束标识
        notifyCloseAsr(ctx, aiPacket);

    }

    private void notifyCloseAsr(ChannelHandlerContext ctx, AiPacket aiPacket) {
        AiPacket aiPacketResponse = new AiPacket();
        aiPacketResponse.setBusinessId(aiPacket.getBusinessId());
        aiPacketResponse.setCommand(Command.QUIT_GROUP_RESPONSE);
        aiServiceContext.CHANNELGROUP.writeAndFlush(aiPacketResponse);
    }

    private void closeAsr(AiPacket aiPacket) {
        SocketClient client = SocketContext.get(aiPacket.getBusinessId());
        client.send("{\"end\": true}".getBytes());
        client.onCloseAwait();
        client.close();
        SocketContext.remove(aiPacket.getBusinessId());
        log.info("\t发送结束标识");
    }

}
```

### 6. 转写的时间延迟问题

连接第三方转写的接口要求：

|   |   |
|---|---|
|内容|说明|
|请求协议|ws[s] (为提高安全性，强烈推荐wss)|
|请求地址|ws[s]: //rtasr.xfyun.cn/v1/ws?{请求参数}  <br>_注：服务器IP不固定，为保证您的接口稳定，请勿通过指定IP的方式调用接口，使用域名方式调用_|
|接口鉴权|签名机制，详见[数字签名](https://www.xfyun.cn/doc/asr/rtasr/API.html#signa%E7%94%9F%E6%88%90)|
|字符编码|UTF-8|
|响应格式|统一采用JSON格式|
|开发语言|任意，只要可以向讯飞云服务发起WebSocket请求的均可|
|音频属性|采样率16k、位长16bit、单声道|
|音频格式|pcm|
|数据发送|建议音频流每40ms发送1280字节|
|语言种类|中文普通话、中英混合识别、英文，小语种以及中文方言可以到控制台-实时语音转写-方言/语种处添加试用或购买|

```
private static void sendMessage(SocketClient client, byte[] fileBytes) throws InterruptedException {
        // 分段处理
        ByteBuffer buffer = ByteBuffer.wrap(fileBytes);
        while (buffer.hasRemaining()) {
            // 处理剩余字节
            if (buffer.remaining() < CHUNCKED_SIZE) {
                byte[] remainingChunk = new byte[buffer.remaining()];
                buffer.get(remainingChunk);
                send(client, remainingChunk);
                // sleep 40ms
                break;
            } else {
                byte[] chunk = new byte[CHUNCKED_SIZE];
                buffer.get(chunk);
                send(client, chunk);
                // sleep 40ms
            }
        }
    }
```

![](https://cdn.nlark.com/yuque/0/2023/png/432307/1703316634680-a67a1d32-c936-4423-b6d9-d75df6828892.png)

因为这个延迟要求，导致我们从前端接收到的数据流必须有每一段的延迟，而随着数据包的分段数越来越大，转写的延迟就会越来越久，导致实时转写非实时。

解决办法：

1. 取消 40ms 的延迟。
2. 采用流水线模式传输数据包。

### 7. 心跳检测

#### 7.1. 概述

网络应用程序普遍会遇到的一个问题：连接假死。连接假死的现象是：在某一端（服务端或者客户端）看来，底层的 TCP 连接已经断开了，但是应用程序并没有捕获到，因此会认为这条连接仍然是存在的，从 TCP 层面来说，只有收到四次握手数据包或者一个 RST 数据包，连接的状态才表示已断开。

连接假死会带来以下两大问题

1. 对于服务端来说，因为每条连接都会耗费 cpu 和内存资源，大量假死的连接会逐渐耗光服务器的资源，最终导致性能逐渐下降，程序奔溃。
2. 对于客户端来说，连接假死会造成发送数据超时，影响用户体验。

通常，连接假死由以下几个原因造成的

1. 应用程序出现线程堵塞，无法进行数据的读写。
2. 客户端或者服务端网络相关的设备出现故障，比如网卡，机房故障。
3. 公网丢包。公网环境相对内网而言，非常容易出现丢包，网络抖动等现象，如果在一段时间内用户接入的网络连续出现丢包现象，那么对客户端来说数据一直发送不出去，而服务端也是一直收不到客户端来的数据，连接就一直耗着。

如果我们的应用是面向用户的，那么公网丢包这个问题出现的概率是非常大的。对于内网来说，内网丢包，抖动也是会有一定的概率发生。一旦出现此类问题，客户端和服务端都会受到影响。

#### 7.2. 解决办法

##### 7.2.1. 服务端空闲检测

我们的服务端只需要检测一段时间内，是否收到过客户端发来的数据即可，Netty 自带的 IdleStateHandler 就可以实现这个功能。

```
pipeline.addLast(new IdleStateHandler(readerIdleTime, writerIdleTime, allIdleTime, TimeUnit.SECONDS));
```

1. 首先，我们观察一下 IMIdleStateHandler 的构造函数，他调用父类 IdleStateHandler 的构造函数，有四个参数，其中第一个表示读空闲时间，指的是在这段时间内如果没有数据读到，就表示连接假死；第二个是写空闲时间，指的是 在这段时间如果没有写数据，就表示连接假死；第三个参数是读写空闲时间，表示在这段时间内如果没有产生数据读或者写，就表示连接假死。写空闲和读写空闲为0，表示我们不关心者两类条件；最后一个参数表示时间单位。在我们的例子中，表示的是：如果 15 秒内没有读到数据，就表示连接假死。
2. 连接假死之后会回调 channelIdle() 方法，我们这个方法里面打印消息，并手动关闭连接。

##### 7.2.2. 客户端定时发送心跳

```
public class MeetPingRequestHandler extends AbstractRequestHandler
    implements MeetRequestHandler<MeetPingMessageRequestPacket> {
    @Override
    public void handler(ChannelHandlerContext ctx, MeetPingMessageRequestPacket meetPingMessageRequestPacket) throws Exception {
        super.addChannelGroup(ctx, meetPingMessageRequestPacket.getGroupId());
        MeetPongResponsePacket meetPongResponsePacket = new MeetPongResponsePacket();
        meetPongResponsePacket.setGroupId(meetPingMessageRequestPacket.getGroupId());
        meetPongResponsePacket.setContent("PONG");
        SessionUtil.getChannelGroup(meetPingMessageRequestPacket.getGroupId()).forEach(channel -> channel
            .writeAndFlush(new TextWebSocketFrame(JSONObject.toJSONString(meetPongResponsePacket))));

    }
}
```

## 优化

1. 与三方的连接改成netty或者直接使用sdk
2. 日志打印
3. 实时翻译
4. 消息可靠性

## 参考文献

1. [实时语音转写 API 文档 | 讯飞开放平台文档中心](https://www.xfyun.cn/doc/asr/rtasr/API.html#%E6%8E%A5%E5%8F%A3%E8%B0%83%E7%94%A8%E6%B5%81%E7%A8%8B)
2. [What is MongoDB?](https://www.mongodb.com/docs/manual/)

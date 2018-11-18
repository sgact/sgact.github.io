---
layout:     post
title:      Netty4\Android6 SSL双向认证攻略
date:       2016-10-13
author:     sg
catalog: true
tags:
    - Android
---


> 本文于16年发表于博客园，现转载自此。文中的实现可能已经过时，但方法应该是通用的。客户端缺少证书校验，仅供参考。

在Android端实现SSL可谓是遍地是坑，出错的原因多种多样，解决方法也各不相同，单单一篇文章不可能填完所有的坑，我会把解决问题的步骤和思路分享给大家。

　　首先要感谢各位前辈的努力，由于SSL调了10来天才调通，中间还隔着10.1，好多文章已经忘了出处，有的还是同事给找的资料，我会尽量在结尾注释出各位前辈的研究。

### 一、Netty JAR包

　　jar包在[Netty的官网](http://netty.io/)，Download处下载，下载到的压缩包中有一个如“all-in-on”的jar。

### 二、SSL密钥

　　所谓双向认证，自然要有服务器和客户端的证书，生成的步骤如下，阅读下面的步骤之后可以大致领悟认证的流程，直接拿来用也是可以的。

　　1.扩展keytool。生成证书要使用jdk中的keytool，直接用keytool生成的密钥是.jks格式，在Android上只能用.bks格式的密钥。为了生成bks格式的密钥，首先要[下载BouncyCastle](http://www.bouncycastle.org/latest_releases.html)，扩展keytool，使他能够生成bks格式的密钥。选择Provider这一列中对应你的jdk版本的jar包。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxbz4s971lj31510afgml.jpg)

    将下载的jar复制到%JRE_HOME%\lib\ext 和 %JDK_HOME%\jre\lib\ext 下

　　然后打开%JRE_HOME%\lib\security\java.security，和%JDK_HOME%\jre\lib\security\java.security\java.security在下图的位置上加一行：
　　security.provider.11=org.bouncycastle.jce.provider.BouncyCastleProvider
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxbz6btyz1j311p0j9q6d.jpg)
注意security.provider.**11**是依次序排下来的

　　另外听说bouncycastle在小米2s~~这个业界毒瘤~~上只有146版本的能用，手上没有测试机，遇到问题的可以改下版本试试。

　　2.生成服务器端的JKS密钥库kserver.keystore

　　keytool -genkeypair -v -alias server -keyalg RSA -sigalg SHA1withRSA -keystore kserver.keystore

　　可以看看[《Java Security：keytool工具使用说明》](http://www.cnblogs.com/f1194361820/p/4266511.html)理解下各个参数的意义，尤其你在开发过程中遇到如SSLHandShake failed之类的错误，一定要根据自己的情景设置各项参数。不能照搬网上的代码。也可以看[Java官方的文档](http://docs.oracle.com/javase/7/docs/technotes/tools/solaris/keytool.html)。下面解释一下两个参数：

　　-sigalg SHA1withRSA

　　-keyalg RSA

　　这两个参数是为了支持Android M，可以在[Android Developer](https://developer.android.com/reference/javax/net/ssl/SSLEngine.html)上找到Cipher suites这张表，有这样一行　
![](https://ws3.sinaimg.cn/large/006tNbRwly1fxbzbnqv0gj30tf013glh.jpg)
后面的数字表示支持的API Level。
现在回过头来看一看这张表，Android N支持的SSL算法又变了，Android N马上就要来了，到时候又要改一次代码。你们如果知道怎么生成Android N的密钥，请务必在评论中告诉我。

　　3.从服务器端密钥库kserver.keystore中导出服务器证书

  ```bash
  keytool -exportcert -v -alias server -file server.cer -keystore kserver.keystore
  ```

　　4.将导出的服务器端证书导入到客户端信任密钥库tclient.bks中，其中客户端信任密钥库自动生成，并且此时要特别指明信任密钥库是BKS类型的

```bash
keytool -importcert -v -alias server -file server.cer -keystore tclient.bks -storetype BKS -provider org.bouncycastle.jce.provider.BouncyCastleProvider
```

　　到这一步的时候留意一下CMD的输出，这里列出了各项密钥信息

　　5.生成客户端密钥库kclient.bks

```bash
keytool -genkeypair -v -alias client -keyalg RSA -sigalg SHA1withRSA -keystore kclient.bks -storetype BKS -provider org.bouncycastle.jce.provider.BouncyCastleProvider
```

　　6.导出客户端证书

```bash
keytool -exportcert -v -alias client -file client.cer -keystore kclient.bks -storetype BKS -provider org.bouncycastle.jce.provider.BouncyCastleProvider
```

　　7.导入生成服务器端信任密钥库
```bash
keytool -importcert -v -alias client -file client.cer -keystore tserver.keystore
```

### 三、随便说一说Netty

　　懂Netty的不用往下看了，因为我也是刚接触

　　1.Netty中的pipeline.

　   可以先网上搜下Netty的大致流程，我就不献丑了。对于pipeline可以看一下《Netty权威指南》的第17章。

　　对于应用了SSL的工程，pipeline的第一个必须是　
```java
pipeline.addLast(new SslHandler(sslEngine));
```
下面是我用到的代码，第一个用于分包。
```java
pipeline.addLast(new LengthFieldBasedFrameDecoder(1024, 4, 4));
pipeline.addLast(new M2MMessageDecoder());
pipeline.addLast(new M2MMessageEncoder());
pipeline.addLast(new ClientHandler());
```
粗略的理解就是把收到的一堆Byte分成若干个HTTP的帧。比较实用的有

- 按行分割　
```java
pipeline.addLast(new DelimiterBasedFrameDecoder(1024, Delimiters.lineDelimiter()));
```

- 按\0分割　
```java
pipeline.addLast(new DelimiterBasedFrameDecoder(1024, Delimiters.nulDelimiter()));
```
这种分割方式要求服务器发送的数据必须要用\0结尾

- 按帧长度分割
```java
pipeline.addLast(new LengthFieldBasedFrameDecoder(1024, 4, 4));
```
这种方式比较灵活，适用于自定义的协议，各参数的意义可以查看[官方文档](https://netty.io/4.0/api/io/netty/handler/codec/LengthFieldBasedFrameDecoder.html)

 　　点开源码可以发现，这三个类都继承于 ByteToMessageDecoder 而之后再pipeline中的ChannelHandler(ChannelHandler表示ChannelPipeline中的各个项，类似与Mina的Filter)都是继承于MessageToMessageDecoder

　　经过了上面的步骤之后pipeline的下一个Channelhandler将以ByteBuf的类型接受到消息数据。

　　要想将ByteBuf解析为POJO，就要定义MessageToMessageDecoder,举个简单的例子，将收到的ByteBuf消息转为byte[]发送给下一个ChannelHandler:
```java
public class M2MMessageDecoder extends MessageToMessageDecoder<ByteBuf> {
    @Override
    protected void decode(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf, List<Object> list) throws Exception {
        int totalByteLength = byteBuf.readableBytes();
        byte[] bytes = new byte[totalByteLength];
        byteBuf.readBytes(bytes);

        list.add(bytes);
    }
}
```

  decode方法中又一个List<Object> list，我们只需向这个list中插入任意格式对象，就完成了解码。样，在下一个ChannelHandler收到这个对象格式的消息。

　　反过来，在encoder中需要把Object转成byte[]

```java
public class M2MMessageEncoder extends MessageToMessageEncoder<byte[]> {
    @Override
    protected void encode(ChannelHandlerContext channelHandlerContext, byte[] bytes, List<Object> list) throws Exception {
        list.add(Unpooled.copiedBuffer(bytes));
    }
}
```
最后一个ChannelHandler用于处理业务逻辑，他继承于SimpleChannelInboundHandler

　　在收到服务器发出的数据时会回调channelRead0方法，如果重写了channelRead方法，channelRead0将不会执行,

注意：
- channelRead0会在收到特定格式数据时被调用（取决于模板）
- channelRead会在收到任意格式时被调用
- 可以添加任意多个ChannelHandler



### 四、给个生成SSLContext的代码
写个文章连点现成的代码都不给，岂不是很没诚意，这个拿去就能用，但仍然有坑。
```java
package com.sg.nettyblackglasses.server;

import android.content.Context;
import java.security.KeyStore;
import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManagerFactory;

public class SslContextFactory
{
    private static final String    PROTOCOL    = "TLSv1.2";//我是坑

    public static SSLContext getClientContext(Context c)
    {
        SSLContext clientContext = null;

        try {
            String keyStorePassword = "1234567";

            // 一定要声明密钥是BKS格式
            KeyStore ks = KeyStore.getInstance("BKS");
            ks.load(c.getResources().getAssets().open("kclient.bks"),
                    keyStorePassword.toCharArray());

            // 这里默认是SunX509
            KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
            kmf.init(ks, keyStorePassword.toCharArray());

            // truststore
            KeyStore ts = KeyStore.getInstance("BKS");
            ts.load(c.getResources().getAssets().open("tclient.bks"),
                    keyStorePassword.toCharArray());

            TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            tmf.init(ts);
            clientContext = SSLContext.getInstance(PROTOCOL);
            clientContext.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return clientContext;
    }
}
```
坑就在这一行
```java
private static final String    PROTOCOL = "TLSv1.2";
```
按理说，这样的代码在Android L/M上是不能正常执行的
![](https://ws4.sinaimg.cn/large/006tNbRwly1fxbzoivrm0j30tc0963z9.jpg)

  这是[developer.android](https://developer.android.com/reference/javax/net/ssl/SSLEngine.html)上的文档，上面明确指出TLSv1.2是不支持API LEVEL 20以下的,也就是4.4及以下，然而在实际使用中却没有任何问题，那么为什么不用TLSv1呢，因为据说TLSv1自身有Bug并不安全。。。虽然暂时没有问题，但是隐约觉得这一定是个坑。唉，反正是只能用它喽。

### 五、未解之谜

  据说Android6要用JDK1.8，然而我用1.7也没什么问题

  javax.net.ssl.SSLHandshakeException: Handshake failed
  　　...

  BAD_DH_P_LENGTH...

  有人遇到这个错误，升级到1.8就好了，然而我是1.7JDK仍然管用，升级JDK这个解决方案看起来相当靠谱，遇到如上错误的不妨试一试，不过上面的错误99%都是密钥没有生成对

  大概就这么多内容了，由于我也是刚接触NIO，对HTTP什么的也不太懂，说错的地方希望各位不要背后笑话我，而是写在评论区，谢谢。

  除了上面文章中的引用，还要感谢以下文章的作者：

- [netty中实现双向认证的SSL连接（有源码哟）](http://blog.csdn.net/virgilli/article/details/42836063)
- [netty源码分析系列文章](http://asialee.iteye.com/blog/1769508)
- 学习Android客户端和服务器端SSLSocket交互的总结 （这个找不到出处了）

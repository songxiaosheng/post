# 从一个 Demo 说起 Zookeeper 服务端源码

# 简介

Zookeeper目前是Apache下的开源项目，作为最流行的分布式协调系统之一，我们平时开发中熟知的Dubbo、Kafka、Elastic-Job、Hadoop、HBase等顶级开源项目中的分布式协调功能都是通过借助Zookeeper来实现的，可以看到想要在生产中保障Zookeeper服务的稳定、快速排查系统问题，深入探究Zookeeper系统的原理是有必要的，那Zookeeper是什么呢？

ZooKeeper 是一个分布式的，开放源码的分布式应用程序协调服务，可用于实现高度可靠的分布式高性能协调的服务，Zookeeper用于维护配置信息，命名，提供分布式同步以及提供组服务的集中式服务，使用Zookeeper可以实现共识，组管理，领导者选举和状态协议，也可以根据自己的需求来构建分布式应用功能。

作为分布式协调系统，Zookeeper并没有直接采用Paxos算法，而是采用一种被称为ZAB（ZooKeeper Atomic Broadcast）的一致性协议。

分布式算法比较抽象枯燥难懂，这里将通过一个内嵌的Zookeeper服务来启动Zookeeper，方便分析调试，从Zookeeper的启动入门，探究一下Zookeeper源码。

# 入门

**依赖**

案例是通过一个Java应用程序来分析的，需要先新建一个Maven项目，然后我们引入Zookeeper的依赖，如下所示：

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.8.0</version>
</dependency>
```

**示例**

入门案例是一个内嵌式的Zookeeper服务，通过运行main方法即可启动Zookeeper服务。

```java
import org.apache.zookeeper.server.ServerConfig;
import org.apache.zookeeper.server.ZooKeeperServerMain;
import org.apache.zookeeper.server.quorum.QuorumPeerConfig;

import java.io.IOException;
import java.util.Properties;

public class EmbeddedZookeeper {

    public static void main(String[] args) throws QuorumPeerConfig.ConfigException, IOException {
        //属性设置
        Properties properties = new Properties();
        //数据目录
        properties.setProperty("dataDir", "/tmp");
        //服务端口
        properties.setProperty("clientPort", "2181");

        //属性转为节点配置
        QuorumPeerConfig quorumPeerConfig = new QuorumPeerConfig();
        quorumPeerConfig.parseProperties(properties);

        //服务端配置
        ServerConfig configuration = new ServerConfig();
        configuration.readFrom(quorumPeerConfig);

        //运行Zookeeper服务
        ZooKeeperServerMain zkServer = new ZooKeeperServerMain();
        zkServer.runFromConfig(configuration);
    }
}
```

代码编写完成之后运行main方法即可启动Zookeeper服务，启动成功之后可以看到如下所示日志，可以看到Zookeeper绑定了2181端口。

![image-20230203082920531](https://static001.geekbang.org/infoq/39/3933d0b721e726344deac663137a7ecd.png)



**客户端连接**

Zookeeper服务启动之后就可以使用客户端来建立连接创建节点了，如下所示abc是我新建的一个节点。

 ![image-20230203083446558](https://static001.geekbang.org/infoq/71/713c4982369549048448b2b527fd1c54.png)



当然也可以输入Zookeeper的四字监控命令如mntr查看Zookeeper服务的一个状态。

![image-20230203083505772](https://static001.geekbang.org/infoq/c0/c02e1456f8985ebfb34a363a135163bd.png)

# 总结

可以看到启动一个单机版的Zookeeper服务并不复杂只需要配置属性，启动服务即可，前面的整个案例仅仅是一个入门案例可以方便借助开发工具来Debug源码，清晰的了解代码执行过程，更多原理与源码解析可以订阅微信公众号  **《中间件源码》**  及时获取。




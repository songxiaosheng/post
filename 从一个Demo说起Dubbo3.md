# 简介

2017年的9月份，阿里宣布重启Dubbo的开发维护，并且后续又将Dubbo捐献给了Apache，经过多年的发展已经发布到3.X版本了，Dubbo重启维护之后是否有值得我们期待的功能呢，下面就来看看吧。

Apache Dubbo 是一款微服务框架，为大规模微服务实践提供高性能 RPC 通信、流量治理、可观测性等解决方案，涵盖 Java、Golang 等多种语言 SDK 实现。

Dubbo3在官网首页的介绍中是这样描述的：Dubbo3是下一代云原生微服务框架 - 3.0 版本的正式发布，标志着 Apache Dubbo 正式进入云原生时代。3.0 在通信协议、服务发现、部署架构、服务治理上都对云原生基础设施进行了全面适配， 提供了Triple、应用级服务发现、Dubbo Mesh等核心特性。

可以看到Dubbo3不再仅仅是一个简单的RPC通信框架了，在云原生实践中Dubbo3逐渐将通用业务下沉与业务逻辑剥离，实现Dubbo Mesh（*Service Mesh*翻译为“服务网格”  *Dubbo Mesh*暂时就翻译为“Dubbo服务网格吧”）功能，如果Dubbo能让这个Mesh功能实现的简单易用，性能高效那微服务项目的开发大家就可以只关注CRUD了。

通过官方社区了解到的资料，Dubbo3 已被阿里巴巴、饿了么、钉钉、工商银行、小米等在生产环境广泛采用。Dubbo3从内到外的推广已经从理论转到了实践，不过是否比原先的Dubbo2好用呢，下面可以从架构到示例来看下。

# 架构

## Dubbo3部署架构

![threecenters](https://cn.dubbo.apache.org/imgs/v3/concepts/threecenters.png)

可以看到Dubbo3的架构图里面新包含了三大中心的概念：

- **注册中心：** 协调 Consumer 与 Provider 之间的地址注册与发现
- **配置中心：** 存储 Dubbo 启动阶段的全局配置，保证配置的跨环境共享与全局一致性，负责服务治理规则（路由规则、动态配置等）的存储与推送。
- **元数据中心：** 接收 Provider 上报的服务接口元数据，为 Admin 等控制台提供运维能力（如服务测试、接口文档等），作为服务发现机制的补充，提供额外的接口/方法级别配置信息的同步能力，相当于注册中心的额外扩展

可以看到这里比较新的概念元数据中心做的事情是用来存储服务接口的元数据，在Dubbo2中服务接口数据与应用数据都是存储在注册中心了，这里将接口元数据抽离到元数据中心，让注册中心专注于应用级服务发现。

## 应用级服务发现模型

![//imgs/v3/concepts/servicediscovery_mem.png](https://cn.dubbo.apache.org/imgs/v3/concepts/servicediscovery_mem.png)



应用级服务发现模型的粒度是以应用单元来进行服务发现的，Dubbo2中的接口级发现不仅仅会造成注册中心数据的膨胀，同时也无法让Dubbo应用于其他微服务框架的应用比如SpringCloud实现互联互通，在打通了地址发现之后，应用级服务发现使得用户探索用 Dubbo 连接异构的微服务体系成为了一种可能。

另外应用级服务发现模型会将元数据信息的获取下沉到一个RPC请求之中，这个让服务通知得以高效，如下图：

![image-20230106083609338](https://a.perfma.net/img/5165651)



Dubbo3的优化不仅仅是这个服务发现模型，不过接口级服务发现模型到应用级服务发现模型的转变是升级过程中最值得关注的其中一个地方。

Dubbo3也有一些其他亮点的功能比如新增基于 HTTP/2 之上定义的下一代 RPC 通信协议Triple 协议,Mesh 解决方案等，感兴趣可以自行去官网查看，这里主要基于一个服务提供者的Demo示例来看下如何使用Dubbo3。

# 服务提供者的Demo

为了更方便了解原理,我们先来编写一个Demo,从例子中来看源码实现:

## 启动Zookeeper

为了Demo可以正常启动,需要我们先在本地启动一个Zookeeper，如下命令所示（Zookeeper可以在官网下载下载之后载conf下新建配置文件zoo.cfg):

```bash
mac@MacdeMacBook-Pro apache-zookeeper-3.6.3-bin % ./bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /Users/mac/Desktop/computer/A/env/apache-zookeeper-3.6.3-bin/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
mac@MacdeMacBook-Pro apache-zookeeper-3.6.3-bin % 
```

前面我们介绍了Dubbo3需要三大中心，这里我们让配置中心，元数据中心和注册中心都使用我们启动的Zookeeper，这样相对方便一些。

## 依赖引入

下面就直接来贴一些核心的依赖内容

```xml
<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.apache.dubbo</groupId>
                <artifactId>dubbo-bom</artifactId>
                <version>3.2.0-beta.3</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
</dependencyManagement>
 <dependencies>
        <!-- dubbo starter -->
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
        </dependency>
        <!-- dubbo Zookeeper注册中心扩展 -->
   			 <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-dependencies-zookeeper</artifactId>
            <type>pom</type>
        </dependency>
      	 <!-- dubbo  开启一些可观测性功能依赖 -->
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-actuator</artifactId>
        </dependency>
</dependencies>
```

## 服务提供者

接下来给大家贴一下示例源码,这个源码来源于Dubbo源码目录的	dubbo-demo/dubbo-demo-api 目录下面的dubbo-demo-api-provider子项目,这里我做了删减,方便看核心代码:
首先我们定义一个服务接口如下所示:

```java
import java.util.concurrent.CompletableFuture;
public interface DemoService {
    /**
     * 同步处理的服务方法
     * @param name
     * @return
     */
    String sayHello(String name);

    /**
     * 用于异步处理的服务方法
     * @param name
     * @return
     */
    default CompletableFuture<String> sayHelloAsync(String name) {
        return CompletableFuture.completedFuture(sayHello(name));
    }
}
```

服务接口对应的实现类如下:

```java
import org.apache.dubbo.rpc.RpcContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.CompletableFuture;

public class DemoServiceImpl implements DemoService {
    private static final Logger logger = LoggerFactory.getLogger(DemoServiceImpl.class);

    @Override
    public String sayHello(String name) {
        logger.info("Hello " + name + ", request from consumer: " + RpcContext.getServiceContext().getRemoteAddress());
        return "Hello " + name + ", response from provider: " + RpcContext.getServiceContext().getLocalAddress();
    }

    @Override
    public CompletableFuture<String> sayHelloAsync(String name) {
        return null;
    }

}
```

## 启用服务提供者

有了服务接口之后我们来启用服务,启用服务的源码如下:

```java
import org.apache.dubbo.common.constants.CommonConstants;
import org.apache.dubbo.config.ApplicationConfig;
import org.apache.dubbo.config.MetadataReportConfig;
import org.apache.dubbo.config.ProtocolConfig;
import org.apache.dubbo.config.RegistryConfig;
import org.apache.dubbo.config.ServiceConfig;
import org.apache.dubbo.config.bootstrap.DubboBootstrap;
import org.apache.dubbo.demo.DemoService;

public class Application {
    public static void main(String[] args) throws Exception {
            startWithBootstrap();
    }
    private static void startWithBootstrap() {
        //创建服务配置
        ServiceConfig<DemoServiceImpl> service = new ServiceConfig<>();
        //为服务配置设置服务接口
        service.setInterface(DemoService.class);
        //为服务配置设置服务接口对应的服务实现
        service.setRef(new DemoServiceImpl());
       //获取Dubbo启动器
        DubboBootstrap bootstrap = DubboBootstrap.getInstance();
        //为Dubbo启动器设置应用配置 这里只设置一个应用名字
        bootstrap.application(new ApplicationConfig("dubbo-demo-api-provider"))
           //为启动器设置注册中心配置（配置中心和元数据可以不用配置模式会选择注册中心的配置）
            .registry(new RegistryConfig("zookeeper://127.0.0.1:2181"))
            //设置RPC通信协议配置为Dubbo协议
            .protocol(new ProtocolConfig(CommonConstants.DUBBO, -1))
            //设置当前Dubbo应用的服务配置
            .service(service)
            //启动Dubbo提供者应用
            .start()
            //等待应用结束
            .await();
    }
}
```

运行main方法启动程序即可。

## 存储在Zookeeper的三大中心节点数据

启动服务,这个时候我们打开Zookeeper图形化客户端来看看这个服务在Zookeeper上面写入了哪些数据,如下图:
![image-20230107202837414](https://a.perfma.net/img/5165673)

这里我们使用同一个Zookeeper集群进行三大中心的数据存储，应用级服务注册主要包含了一个服务名字和IP端口信息，元数据信息主要包含具体的服务接口信息。

如果了解Dubbo2.X老版本的同学应该可以看到原先Dubbo低版本的接口级的服务数据被拆分为了多个部分：元数据、应用级数据、配置数据。细化后的服务数据拆分看似复杂了其实有效的解决了传统接口级服务发现模型的性能存储问题。

在这里通过服务提供者的示例简单介绍了一下Dubbo3,可以先大致了解下,在后面会有更详细的Dubbo3源码解析系列来剖析Dubbo3源码， 通过透析代码来看透Dubbo3服务注册原理,服务提供原理, 如果感兴趣可以关注微信公众号 **《中间件源码》** 订阅后续内容吧。

# 简介

对于大部分开发人员来说可能用过普罗米修斯Grafana这样的监控系统，从未听说过Micrometer工具，这里就详细的来介绍下可观测性神器Micrometer，让你在开发时使用它就和使用SLFJ 日志系统一样简单易用，有效的提升系统的健壮性和可靠性。

# 可观测性

在了解Micrometer之前可以先来简单了解下云原生微服务时代下人人追捧的可观测性概念，这会更有利于我们理解Micrometer的作用，在传统单体应用时代对于服务的检查和诊断可以借助于简单的报表，监控和日志就可以有效的解决，而现在为了易于分工，提升开发变更效率，隔离故障等需求驱动下，一个系统往往被拆分成多个服务我们称它们为微服务，另外近几年火热的云原生，ServiceMesh，Serverless等概念更是打算在基础设施层做变革进行降本增效，可以看到一个相对简单的单体系统已经变得非常复杂，想要了解下内部运行健康状况如何是比较困难的，出现问题的时候也往往让人摸不着头脑。这时候就有人提出了可观测性的概念，可观测性是个比较大的概念就像是我们开发人员有了透视能力一样一眼可以看穿系统的内部运行状况，当然这是不现实的。所以就有人进行了对它进行了具体的定义，广为流传又易于理解的说法是可观测性的三大支柱说:Metrics、Tracing、Logging 。

## 可观测性三大支柱说

可以看到中医有“望闻问切”的方式来诊断病人的病症，我们有了可观测性三大支柱Metrics、Tracing、Logging来帮助我们了解和排查系统运行健康状况。

- **Logging(日志) ：** 日志大家是最熟悉的也是是最容易生成的，比如大家学习输出到控制台的Hello World,Java通过log4j打印的日志,日志虽然容易生成但是分析汇总处理起来比较麻烦往往经过多层转换性能非常底下，

- **Tracing （追踪）：** *追踪* 是一系列因果相关的分布式事件的表示，这些事件对通过分布式系统的端到端请求流进行编码，通过保留因果关系来进行回顾性分析和故障排除，使开发人员能够更好地了解请求的生命周期。关于链路追踪小编了解到的比较核心的方式一般服务在跨系统的调用时想要将其串起来就需要用到traceId传递，在内部线程之间流转就需要用到SpanId，如果拿到异常的追踪ID就可以快速的定位相关的位置，链路追踪的麻烦之处就是需要改造现有系统让所有的位置支持，不过关于全链路系统国内外都要比较完善的开源中间件来解决，比如Zipkin和Jaeger是两个最流行的OpenTracing兼容开源分布式跟踪解决方案。

- **Metrics(指标)：** *指标* 是在时间间隔内测量的数据的数字表示。指标可以利用数学建模和预测的力量来获取系统在当前和未来一段时间内的行为知识。指标一旦被收集，就更容易进行数学、概率和统计转换，例如采样、聚合、汇总和关联。这些特征使指标更适合报告系统的整体健康状况，由于指标一般是我们处理过的数据更为精确所以更适合用于监控分析，触发警报。所以一般业务开发同学接触指标相对较少，平时大部分应用层和基础设施层的监控指标埋点一般都会由负责监控的同学完成，不过为了系统更健壮系统开发的时候要尝试在可能出现问题的地方进行多多埋点。

可以看到可观测性的三大支柱在不同的维度提供支持使系统更易于观察，理论性的概念可能不太明显，这里可以给举一个借助客观性理论排查请求超时的场景（当然实际情况可能比这个复杂的多），如果系统在预先对某个服务消费者和生产者请求进行了日志打印，追踪处理，埋点监控，当发生了请求调用失败的时候埋点监控的将异常告警给我们可以及时发现问题，然后打开链路追踪系统排查具体出现问题的系统，拿到链路追踪ID之后可以打开日志根据链路追踪ID查询到所有相关的日志来排查出具体原因。可以看到如果遇到问题的时候不要急于打开日志，通过综合分析各项指标，追踪排查，最后得出结论，就像是中医的望闻问切一样。善于利用这些工具可以有效的帮助我们解决项目中常见的问题，可以联想下平时遇到的问题是不是大部分情况只要掌握了足够的信息就可以解决，可观测性的三大支柱的排查问题这个场景的使用总结成一句话就是：监控埋点发现问题-> 链路追踪定位问题-> 日志和工具解决问题。

# Micrometer

## Micrometer简介

前面长篇大论的描述了一下可观测性，这个时候应该就可以了解指标埋点所处的位置，在何处何时帮助我们发现问题。接下来就正式进入本文的主题Micrometer开源组件。

官方式是这样介绍的：Micrometer为最流行的监控系统提供了一个简单的仪表客户端外观，允许您在没有供应商锁定的情况下基于JVM的应用程序代码进行仪表化。

可以想象一下大家熟悉的SLF4J日志客户端门面，Micrometer其实就是一个监控埋点的客户端门面。

## 为什么要使用Micrometer？

自己埋点当然也可以既然有现成的工具又何必造轮子，下面可以看下Micrometer的特性：

- **多维度指标：** 默认提供计时器**、**仪表**、**计数器**、**分布摘要和长任务计时器等指标与中立的接口。

- **丰富的指标绑定binder：** 开箱即用的缓存检测、类加载器、垃圾收集、处理器利用率、线程池，以及为可操作的洞察量身定制的更多工具，我们也可以自行扩展开发自己的绑定指标工具。

- **集成到Spring中：** Spring Boot 应用程序交付应用程序默认的指标的检测库，其他项目集成也仅仅需要一两个一两个依赖即可。

- **支持流行的监控系统：** 作为检测门面外观，Micrometer 允许您使用供应商中立的接口使用维度指标检测代码，并在最后一步决定监控系统。使用 Micrometer 检测您的核心库代码允许将库包含在将指标发送到不同后端的应用程序中。包含对AppOptics、Azure Monitor、Netflix、Atlas、CloudWatch、Datadog、Dynatrace、Elastic、Ganglia、Graphite、Humio、 Influx /Telegraf、JMX、KairosDB、New Relic、Prometheus、SignalFx、Google Stackdriver、StatsD和Wavefront的内置支持。



## 开发入门

### 依赖

Micrometer 包含一个带有检测 SPI (Service Provider Interface 一种扩展机制）的核心库和一个不将数据导出到任何地方的内存中实现，一系列具有各种监控系统实现的模块，以及一个测试模块。这里依赖主要介绍两个一个是核心依赖，一个是适配第三方监控的扩展依赖。

- 核心依赖为：micrometer-core 核心的注册表，监控指标，默认提供的绑定配置都在这里。

  ```xml
     <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-core</artifactId>
      <version>${version}</version>
      <scope>compile</scope>
    </dependency>
  ```

- 适配第三方监控的依赖：用于将指标适配到第三方监控系统，比如我们要将指标输送给普罗米修斯prometheus监控系统就需要引入依赖micrometer-registry-prometheus 

  ```xml
   <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
        <version>${version}</version>
        <scope>compile</scope>
    </dependency>
  ```

完整的依赖引入方式如下图所示：

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.10.2</version>
    <exclusions>
        <exclusion>
            <artifactId>micrometer-core</artifactId>
            <groupId>io.micrometer</groupId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-core</artifactId>
    <version>1.10.2</version>
</dependency>
```



### 入门示例

#### 例子

为了便于理解我们先来从一个简单的示例中来看，下面就来直接贴代码：

```java
import com.sun.net.httpserver.HttpServer;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.composite.CompositeMeterRegistry;
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import io.micrometer.prometheus.PrometheusConfig;
import io.micrometer.prometheus.PrometheusMeterRegistry;

import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;

/**
 * Hello world!
 *
 */
public class MicrometerTest
{
    public static void main( String[] args )
    {
      	//组合注册表
        CompositeMeterRegistry composite = new CompositeMeterRegistry();
				//内存注册表
        MeterRegistry registry = new SimpleMeterRegistry();
        composite.add(registry);
				//普罗米修斯注册表
        PrometheusMeterRegistry prometheusRegistry = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
        composite.add(prometheusRegistry);
				//计数器
        Counter compositeCounter = composite.counter("counter");
        //计数
        compositeCounter.increment();
        try {
            //暴漏8080端口来对外提供指标数据
            HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
            server.createContext("/prometheus", httpExchange -> {
                //获取普罗米修斯指标数据文本内容
                String response = prometheusRegistry.scrape();
                //指标数据发送给客户端
                httpExchange.sendResponseHeaders(200, response.getBytes().length);
                try (OutputStream os = httpExchange.getResponseBody()) {
                    os.write(response.getBytes());
                }
            });

            new Thread(server::start).start();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```



通过运行main方法，在浏览器中访问地址http://localhost:8080/prometheus 就可以看到我们程序提供给普罗米修斯监控的指标了

![image-20221203114539498](images/meter.png)

这个例子相对还是比较简单的，可以动手尝试一下，便于理解。

#### 指标注册表MeterRegistry

可以看到我们最终想要获取的数据其实就是一个一个的Meter(指标）数据，Meter(指标）是用于收集应用程序的一组测量值，Meter(指标）在Micrometer中有单独指标接口类型为Meter。Micrometer 中的Meter是从MeterRegistry指标注册表中创建的（一般不是由我们自行创建的注册表会进行注册缓存等各种操作我们只需要调用它的方法来创建即可）. 每个支持的监控系统都有一个MeterRegistry. 注册表的创建方式因每个实现而异，下面可以看下上面例子中出现的几个注册表：

- **内存注册表SimpleMeterRegistry：**micrometer-core依赖内部包括一个`SimpleMeterRegistry`在内存中保存每个仪表的最新值并且不会将数据导出到任何地方。如果还没有首选的监控系统，可以使用简单的注册表开始使用指标，数据在内存中可以自行管理。

- **组合注册表CompositeMeterRegistry：**micrometer-core依赖中提供了一个`CompositeMeterRegistry`可以添加多个注册表的工具，让您可以同时将指标发布到多个监控系统，如果只有组合注册表其实意义并不是很大，组合注册表不会创建实际存在的指标，组合注册表其实主要用来将各个注册表聚合起来的。
- **普罗米修斯注册表PrometheusMeterRegistry ：**当使用普罗米修斯监控时，引入的micrometer-registry-prometheus这个依赖中提供了一个`PrometheusMeterRegistry`用于将指标数据转换为普罗米修斯识别的格式和导出数据等功能。

在SpringBoot程序中已经集成好了这个注册表，可以尝试找一找SpringBoot程序有哪些可用的注册表。

#### 指标Meter

前面简单介绍了下其实我们整个过程都是围绕着Meter(指标）的，Micrometer内部需要处理各种指标Meter来进行度量程序，我们最终想要获取的数据其实就是一个一个的Meter(指标）数据来进行诊断应用的健康状况，Micrometer 支持一组`Meter`原语，下面是一些常见的指标类型：

- **`Timer` （计时器）：** 用于测量短时延迟和此类事件的频率。

- **`Counter` （计数器）** ：计数器记录单一计数指标，该`Counter`接口允许按固定数量递增，该数量必须为正数，可以用来统计无上限的数据。

- **`Gauge`  （仪表盘）：** 一般用来统计有上限可增可减的数据，仪表是获取当前值的句柄。仪表的典型示例是集合或映射的大小或处于运行状态的线程数。

- **`DistributionSummary`（分布摘要跟踪事件的分布）：** 它在结构上类似于定时器，但记录的是不代表时间单位的值。例如，您可以使用分布摘要来衡量到达服务器的请求的负载大小。

- **`LongTaskTimer`（长任务计时器）：** 长任务计时器是一种特殊类型的计时器，可让您在正在测量的事件**仍在运行**时测量时间。一个普通的 Timer 只记录任务完成后的持续时间。

- **`FunctionCounter`（函数计数器）：** 在函数编程中可以传递一个函数，在需要时调用函数进行获取数据。

- **`FunctionTimer`（函数计时器）：** 在函数编程中可以传递一个函数，在需要时调用函数进行获取数据。

- **`TimeGauge`（跟踪时间值的专用量规）：** `TimeGauge`是一个跟踪时间值的专用量规，可缩放到每个注册表实现所期望的基本时间单位。

不同的仪表类型会产生不同数量的时间序列指标。例如，虽然只有一个指标表示 a `Gauge`，但 a 可以`Timer`衡量定时事件的计数和所有定时事件的总时间。

# 总结

可以看到Micrometer封装了一套标准的可观测性指标类型，并且提供了基础的注册表帮助生成与临时存储指标数据，如果要将指标数据输送到监控系统仅仅需要额外引入一个适配第三方监控系统的扩展包即可。更好用的是SpringBoot的指标埋点都是基于Micrometer实现的，并且已经内置了很多指标绑定器用于绑定指标，比如JVM、Tomcat、Http等等，感兴趣的小伙伴可以尝试开启试试吧，增加一些有用的指标埋点让程序更透明, 对Micrometer感兴趣可以关注微信公众号 **《中间件源码》** 订阅交流吧。



# 简介

对于Log4j2大家应该都不是很陌生，听说最多的应该是2021年年底出现的安全漏洞了，不过最让大家头痛的应该不仅仅是这个安全漏洞的处理，安全漏洞通过升级最新的依赖版本即可快速解决，平时在使用过程中遇到过比较多的问题应该就是日志jar包不知道如何选择？日志jar冲突引起的日志不打印问题，日志配置太过复杂不知道如何配置只能百度CV粘贴一个配置。这些日志配置其实并不复杂，主要是因为日志组件的发展历史比较充满曲折，导致了很多地方不兼容。接下来就来通过日志组件的发展历史来入手，看看Log4j2是从什么背景下产生的。

## 历史

Log4j2日志出现的这些问题多少与它出现的历史有点关系，接下来就先来了解下Java日志发展史，方便我们后续知道引入哪个依赖组件。

### System.out

对于Java日志打印最开始只有大家熟悉的以System开头如System.out.println("hello world")这样的写法，默认的控制台日志打印方式需要有IO操作,性能极其低效(慎用)，功能也太过单一只能简简单单的输出日志。

### Log4j

再后来就有了软件开发者Ceki Gulcu设计出了一套日志库也就是log4j并捐献给了Apache帮助简化日志打印。相关的依赖包是log4j和适配log4j2的桥接包log4j-1.2-api。

### JUL（Java Util Logging）

Java毕竟还是sun公司(后被oracle收购)的Java,sun公司并没有采纳Log4j提供的标准库，而是在jdk1.4自立一套日志标准JUL（Java Util Logging) JUL并不算强大也没得到普及所以现在大家也很少听说了。相关的依赖是jdk和适配log4j2的桥接包log4j-jul

### JCL （Jakarta Commons Logging）

为了方便开发者进行选择使用，Apache推出了日志门面JCL（Jakarta Commons Logging）可以在运行时绑定日志组件。 相关的依赖是commons-logging和适配log4j2的桥接包log4j-jcl。 

### Slf4j

前面的竞争促进了日志组件的发展但也逐渐导致日志依赖与配置越来越复杂，2006年Log4j的作者Ceki Gulcu离开了Apache组织后觉得JCL门面不好用，于是自己开发了一版和其功能相似的Slf4j（Simple Logging Facade for Java）这个也是大家所熟悉的日志门面。相关的依赖是slf4j-api和适配log4j2的桥接包og4j-slf4j-impl或者log4j-slf4j2-impl。

### Logback

后来Slf4j作者又写出了Logback日志标准库作为Slf4j接口的默认实现。

### Log4j2

到了2012年，Apache可能看到这样下去要被反超了，于是就推出了新项目Log4j2并且不兼容Log4j，全面借鉴Slf4j+Logback。Apache Log4j 2是对Log4j的升级，它比其前身Log4j 1.x提供了显著的改进，并提供了Logback中可用的许多改进，同时修复了Logback体系结构中的一些固有问题。

了解了日志组件的历史，可以看到最后log4j2集众家之长，那应该如何优雅的使用log4j2日志呢，可以继续往下看。

# 架构说明

## 定位

Log4j 2 旨在用作审计日志记录，被设计为可靠、快速和可扩展，易于理解和使用的框架。简单的来说Log4j2就是一个日志框架，用来管理日志的。

## 特征

之所以要使用Log4j2 主要还是因为Log4j2 为我们提供了足够好用的支持，下面可以来看下Log4j2的一些特征：

- **API分离：**  API 与实现是分开的。
- **改进的性能：** 支持Disruptor 异步日志记录。
- **支持多种API：** 提供对 Log4j 1.2、SLF4J、Commons Logging 和 java.util.logging (JUL) API 的支持。
- **无侵入性：** 通过扩展机制自动加载，无需与代码完全耦合，代码中可以使用SLF4J门面
- **插件架构：** 插件化配置， 自动识别插件并在配置引用它们，极高的可扩展性
- **属性配置支持：** 可以在配置中引用属性，Log4j 将直接替换它们，属性来自配置文件中定义的值、系统属性、环境变量、ThreadContext Map 和事件中存在的数据。
- **无垃圾与低垃圾** ：稳态日志记录期间，Log4j 2在独立应用程序中是无垃圾的，Web 应用程序中是低垃圾的。

可以看到Log4j2 核心的机制中考虑到了高性能，可扩展，可配置等需求，有效的解决着我们使用日志的痛点，那接下来就来从整体来了解下Log4j2。

## 架构

下面可以先整体来了解下UML图，这里我用文字的形式标明了日志类型的作用，可以简单了解下。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2ba309a1179c4b878f0e9fa3619a9297.png#pic_center)



如果对UML不是非常熟悉的同学看起来可能会比较费劲，不过不用担心下面就针对比较重要的类型详细来说明下，一方面便于了解日志配置，一方面便于后面我们使用。

- **LoggerContext(日志上下文) ：**   这个就像是Spring的ApplicationContext 充当着容器的上下文环境，Spring可以同时存在应用上下文，Web上下文，Log4j2应用也可以同时有多个 LoggerContext
- **Configuration （配置）：** 每个 LoggerContext 都有一个活动的 Configuration。Configuration 包含所有 Appenders、context-wide Filters、LoggerConfigs 并包含对 StrSubstitutor 的引用
- **Logger（记录器）：**  用于让使用者打印日志使用，可以为每个类创建不同的日志记录器，Logger 本身不执行任何直接操作。它只有一个名称并与 LoggerConfig 相关联由日志实现根据配置来进行打印日志。
- **LoggerConfig（记录器配置**）：  LoggerConfig对象是在日志记录配置中声明Logger时创建的。LoggerConfig包含一组筛选器Filter，这些筛选器必须允许LogEvent在传递给任何Appender之前通过。它包含对应用于处理事件的一组Appender的引用。
- **Log Levels （日志级别）：** LoggerConfigs 将被分配一个 Log Level 内置级别集包括 ALL、TRACE、DEBUG、INFO、WARN、ERROR、FATAL 和 OFF。Log4j 2 还支持自定义日志级别 ，下表说明了级别过滤的工作原理。在表中，垂直标题显示 LogEvent 的级别，而水平标题显示与适当的 LoggerConfig 关联的级别。交集标识是否允许 LogEvent 通过以进行进一步处理 (Yes) 或丢弃 (No)。

- **Filter（筛选器）：** 除了如上一节所述发生的自动日志级别过滤之外，Log4j 还提供了 Filter，可以在控制权传递给任何 LoggerConfig 之前、在控制权传递给 LoggerConfig 之后但在调用任何 Appender 之前、在控制权被执行之后应用。
- **Appender（追加器）：** Log4j 允许记录请求打印到多个目的地。在 log4j 中，输出目的地称为 Appender。多个 Appender 可以附加到一个 Logger。目前，存在用于控制台、文件、远程套接字服务器等日志的追加
- **Layout（布局）：** 通常情况下，用户不仅希望自定义输出目标，还希望自定义输出格式。这是通过将 Layout 与 Appender 相关联来实现的。Layout 负责根据用户的意愿格式化 LogEvent，而 appender 负责将格式化的输出发送到其目的地。PatternLayout是标准 log4j 发行版的一部分，它允许用户根据类似于 C 语言printf函数的转换模式指定输出格式。
- **StrSubstitutor 和 StrLookup：** StrSubstitutor 类和 StrLookup 接口是从 Apache Commons Lang借来的，然后进行了修改以支持评估 LogEvents。另外 插值器 类是从 Apache Commons Configuration 借来的，以允许 StrSubstitutor 评估来自多个 StrLookups 的变量。它也被修改为支持评估 LogEvents。这些一起提供了一种机制，允许配置引用来自系统属性、配置文件、ThreadContext Map、LogEvent 中的 StructuredData 的变量。如果组件能够处理变量，则可以在处理配置时或在处理每个事件时解析变量。

简单的了解了Log4j2的一些概念之后可能并不是很容易理解一些概念的具体含义，使用起来可能还会比较费劲，那接下来就通过一个简单又完整的入门例子来看下.

# 开发入门

为了增加一点点的难度，也贴近一下平时开发使用的诉求，这里就以Log4j2绑定Slf4j的案例来说明,使用Slf4j来作为日志门面，使用Log4j2来实现具体的日志配置与打印。

同时下面的示例会有这样的需求：

- **错误日志打印：** 将error日志级别的日志额外打印到error.log里面方便问题排查。
- **业务日志打印：**  将位于link.elastic包及其子包下的所有日志打印到logger.log日志里面。
- **非业务日志打印：** 如果不满足link.elastic的包的日志则打印到控制台。
- **日志归档：** 所有的日志文件都要具有归档策略比如按日期每天归档，或者文件超过250MB也要归档。
- **链路追踪Id打印：** 详细的日志打印可以在Java代码中设置链路追踪Id TraceId打印日志的时候可以将其打印出来。

下面就来详细看下满足这样5个需求的日志配置是如何实现的吧。

## 依赖引入

可以先通过如下图来看下Log4j2与Slf4之间的适配需要引入哪些依赖包：

![Diagram showing which JARs correspond to which systems](https://logging.apache.org/log4j/2.x/images/whichjar-2.x.png)

可以看到如果要使用Slf4j门面的话，需要引入一个Slf4j门面依赖包slf4j-api和一个与log4j2绑定slf4j的桥接包log4j-slf4j-impl，下面就来看下我们要引入的依赖：

```xml
 <dependencyManagement>
    <dependencies>
       <!--通过BOM物料清单来引入log4j2的版本-->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-bom</artifactId>
            <version>2.19.0</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>
     </dependencies>
</dependencyManagement>
<dependencies>
    <!--log4j2 API包-->
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-api</artifactId>
    </dependency>
    <!--log4j2核心实现包-->
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
    </dependency>
    <!--用于log4j2与slf4j桥接-->
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-slf4j-impl</artifactId>
    </dependency>
    <!-- slf4j门面包-->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.25</version>
    </dependency>
</dependencies>
```

## 入门示例

### 日志配置log4j2.xml

在Log4j2中日志的配置文件是大部分情况下是通过配置日志的xml文件来生效的，这个配置文件的路径默认是在类的根路径下的log4j2.xml配置文件中，当然也可以通过在JVM参数中指定一个其它位置的日志配置路径，具体参数配置的KEY为log4j.configurationFile，接下来就在maven项目的根目录src/main/resources目录下创建一个log4j2.xml配置文件来让配置默认生效，具体配置内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
    <!--属性配置-->
    <Properties>
        <!--日志打印格式-->
        <Property name="PATTERN">%date{HH:mm:ss.SSS} [%thread] %X{TraceId} %-5level %logger{36} - %msg%n</Property>
        <!--日志等级-->
        <Property name="LEVEL">INFO</Property>
    </Properties>
    <!--日志追加器配置-->
    <Appenders>
        <!--可滚动归档文件的日志追加器，这里配置的是Error级别的日志可以打印到error.log文件中
        同时根据日期（天）和大小（最大250MB)进行文件归档-->
        <RollingFile name="ErrorFile" fileName="error.log"
                     filePattern="$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
            <!--日志文件打印格式-->
            <PatternLayout>
                <Pattern>${PATTERN}</Pattern>
            </PatternLayout>
            <!--日志文件归档策略-->
            <Policies>
                <!--根据日期格式按时间归档-->
                <TimeBasedTriggeringPolicy/>
                <!--根据文件大小归档超过250MB就归档-->
                <SizeBasedTriggeringPolicy size="250 MB"/>
            </Policies>
            <!--日志过滤器-->
            <Filters>
                <!--阈值过滤器，日志等级大于等于ERROR的接收其他的都拒绝-->
                <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMi**atch="DENY"/>
            </Filters>
        </RollingFile>
        <!--日志logger文件可以接收所有级别的日志打印-->
        <RollingFile name="LoggerFile" fileName="logger.log"
                     filePattern="$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
            <!--日志布局 -->
            <PatternLayout>
                <Pattern>${PATTERN}</Pattern>
            </PatternLayout>
            <!--归档策略-->
            <Policies>
                <!--根据日期格式按时间归档-->
                <TimeBasedTriggeringPolicy/>
                <!--根据文件大小归档超过250MB就归档-->
                <SizeBasedTriggeringPolicy size="250 MB"/>
            </Policies>
        </RollingFile>
        <!--打印日志到控制台的日志追加器-->
        <Console name="STDOUT" target="SYSTEM_OUT">
            <!--日志布局格式-->
            <PatternLayout pattern="${PATTERN}"/>
        </Console>
    </Appenders>
    <!--日志记录器配置-->
    <Loggers>
        <!-- 记录器的日志名字，这个日志记录器的名字与我们每个类里面获取的Logger对象对应，
        对应的关系就是通过这个name来匹配的，匹配规则一般是满足Logger配置的name前缀，
        每个logger元素的日志上下文中都存在一个LoggerConfig配置对象来管理配置-->
        <Logger name="link.elastic" additivity="false">
            <!-- LoggerConfig 也可以配置一个或多个 AppenderRef 元素，
            在处理日志记录事件时将调用它们中的每一个-->
            <!--logger.log文件日志追加器-->
            <AppenderRef ref="LoggerFile"/>
            <!--error级别的error.log追加器-->
            <AppenderRef ref="ErrorFile"/>
        </Logger>
        <!--  每个配置都必须有一个根记录器。前面的Logger日志配置器未匹配到则走默认的根记录器
        如果未配置默认根 LoggerConfig，其级别为 ERROR 并附加了控制台附加程序，将被使用。
        根记录器和其他记录器之间的主要区别是:
        1.根记录器没有名称属性。
        2.根记录器不支持可加性属性，因为它没有父记录器-->
        <Root level="${LEVEL}">
            <!--控制台追加器-->
            <AppenderRef ref="STDOUT"/>
        </Root>
    </Loggers>
</Configuration>
```

为了全面的了解我们前面介绍了一些日志相关的概念，这里在引入日志配置的时候尽可能的关联到更多的元素,引入了日志配置之后，下面可以来看Java代码打印日志的示例，同时看下打印效果方便理解。

### Java打印日志示例

#### 位于link.elastic包下包含main函数的启动类源码如下：

```java
package link.elastic.biz;

import com.demo.DemoLog;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

/**
 * Hello world!
 */
public class App {
    static Logger logger = LoggerFactory.getLogger(App.class);

    public static void main(String[] args) {
        //设置日志上下文MDC
        MDC.put("TraceId", "123456");
        logger.debug("Hello World!");
        logger.info("Hello World!");
        logger.warn("Hello World!");
        logger.error("Hello World!");
        new DemoLog().log("Hello World!");
        MDC.clear();
    }
}
```

#### 位于其他（非link.elastic）包下的DemoLog日志打印示例：

```java
package com.demo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class DemoLog {
    Logger logger = LoggerFactory.getLogger(DemoLog.class);

    public void log(String s) {
        logger.debug(s);
        logger.info(s);
        logger.warn(s);
        logger.error(s);
    }
}
```

### 日志打印效果

#### 控制台日志（非link.elastic包的日志）

 ![logger.png](https://a.perfma.net/img/5154768)

#### logger.log中的日志（link.elastic包下的日志）

![logger.png](https://a.perfma.net/img/5154753)

#### error.log中的日志

 ![errror.png](https://a.perfma.net/img/5155733)
可以看到这个例子充分的满足了前面的5大诉求：

- **错误日志打印：**  这里使用了阈值过滤器ThresholdFilter，日志等级大于等于ERROR的接收打印其他的都拒绝
- **业务日志打印：**  这里我们单独配置了日志记录器Logger并将其name属性设置为了link.elastic只要Java代码中的日志记录器满足前缀为link.elastic就会将日志打印到这个文件里面，在Java代码中我们的日志记录器的名字为link.elastic.biz.App 是满足link.elastic的前缀的所以会将日志打印到logger.log里面。
- **非业务日志打印：** 对于不满足link.elastic的包比如这里的包名为com.demo下的日志是无法匹配到前面业务日志打印的日志记录器的就只能走Root这个根日志记录器，这个根日志记录器的追加器配置的是控制台，前面控制台打印的日志就是非link.elastic包下的日志打印。
- **日志归档：** 这里可能没有很明显的展示因为要满足日期格式或者大小，日期归档使用的是TimeBasedTriggeringPolicy 这个策略根据filePattern中的日期来进行归档最小的时间我们设置的是日会再每天0点之后产生新日志的时候进行归档，大小归档设置的是SizeBasedTriggeringPolicy。
- **链路追踪Id打印：** 对于链路追踪系统往往不仅仅会将链路信息输送到第三方链路追踪系统也会将链路信息打印控制台一份， 这里我们使用的是字符串替换器，在日志打印格式中设置获取链路追踪id的获取方式%X{TraceId} ，然后在Java代码中将链路追踪Id放入日志诊断上下文MDC中即可如代码： `MDC.put("TraceId", "123456");`

# 总结

日志也是我们最常用的观测系统健康状况的方式，优雅的日志打印可以在排查问题的时候事半功倍，在Java日志组件中很多地方使用了日志实现自动扫描的扩展机制，如果随意引入不兼容的依赖包之后被扩展机制扫描到，就很容易出现日志不打印的问题，对于Java 日志依赖的引入，我们可以先了解其曲折的发展历史，了解其依赖来源，然后再针对性的配置依赖即可。然后就是log4j2日志的配置，关于日志的配置官网有非常详细的文档，在使用的时候CV了百度下来的日志配置之后可以参考官网详细的配置，尝试自定义各种属性比如日志追加器append针对日志进行指定位置输出，归档、日志记录器logger针对日志进行分层处理等。如果还有其他问题可以关注微信公众号 **《中间件源码》** 一起交流吧。

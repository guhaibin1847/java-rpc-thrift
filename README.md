# java-rpc-thrift
## 简单介绍
这是一个简单小巧的Java RPC框架，适用于Java平台内、为系统之间的交互提供了、高性能、低延迟的方案。适合在集群数量偏少的情况下使用（50台以下集群环境）。当然、它也可以在大型集群环境下使用，由于未引入Zookeeper支持，所以它在大型集群环境下不够成熟，例如服务发现以及监控都没有做，但是作为RPC框架来用已经足够，至少比使用rest、webservice等性能高得多，也比直接使用thrift、avro等方便的多。

为了让它保持小巧、简单，所以不打算引入Zookeeper支持。我认为50台server组成的集群，已经可以满足绝大部分需求，所以简单、小巧、高性能才是最重要的。如果你认为简单不重要，或者成熟度是最重要的，那么淘宝的Dubbo在等着你。http://dubbo.io/

## 背景以及需求
心血来潮，在公司很无聊，这才是主要原因。 其次是我们要对系统模块进行拆分，从原系统移出，那么就需要寻找一个远程调用工具。
1、基于HTTP（rest、webservice等） 主要性能很差，其次是难以支持高并发，而且组装HTTP请求也比较麻烦，难以形成规范，所以此项被pass。
2、基于thrift，性能虽好，但是使用起来非常麻烦，需要频繁的生成代码，而且对Client开发者要求较高，需要自己写连接池，长连接无法使用LVS还要写负载均衡和容错。而且thrift的服务端需要将业务逻辑全部放在一个接口（一个接口就需要发布一个服务，占用一个线程池），这将是个很恶心的事，所以也被pass。

正因为以上两点，所以我打算自己写一个框架。要求是：简单小巧、依赖少、高性能、高并发、支持集群、负载均衡、容错。无学习成本，源码简单可定制修改，我认为这些才是最主要的。

如果你也在寻找这样一个框架，那么很值得看一下。

## 框架详解
我做完了这个框架，没多久便发现了Dubbo，在看了Dubbo的设计以后，惊喜的发现此框架和Dubbo的核心功能几乎一样。

说起来很简单，就是框架会在Client端代理一个接口，调用这个接口的方法，将发送远程请求，参数序列化传递到远程Server端，Server端处理业务逻辑，完成后、将返回结果序列化给Client端，作为被调用方法的返回值。因此整个过程对用户是透明的。

项目底层使用thrift，这是为了使用thrift的各种Server模型，以此来支持高并发，低延迟。没有使用Netty，原因是Netty较重，延迟要比thrift稍高一些，Netty适合处理高吞吐的异步IO，对于RPC的同步调用没有好处。Netty并不适合。您不用担心thrift性能有问题，也不用担心thrift框架太重，我做过测试，性能和直接使用soket通信几乎不相上下，thrift框架的代码特别少，仅仅是对soket的简单封装，框架非常轻便。

序列化工具使用kryo，这也是性能的关键，您也可以自己去查一下kryo相关资料，这里就不说他了，序列化结果很小，速度很快就是了。

框架依赖 thrift、kryo、commons-pool、spring-beans（其中kryo可以自行替换为您喜欢的序列化工具）

集群支持随机负载均衡，轮询负载均衡（您也可以自己写负载均衡实现），优雅停机(kill pid不要加-9)，容错（集群某几台挂掉并不影响服务）

线程模型 以ThriftThreadPoolServer、ThriftTThreadedSelectorServer 两种为主，具体细节参考thrift（您也可以自己实现Server）

## 使用例子
### 服务端：
/appchina-rpc-test/src/main/java/main/Server.java

ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("/application-server.xml");
ctx.registerShutdownHook();
ctx.start();

/appchina-rpc-test/src/main/resources/application-server.xml

<bean id="servicePublisher" class="com.appchina.rpc.thrift.remote.base.ThriftServicePublisher">
    <property name="definitions">
        <!-- 需要发布的服务列表 -->
        <array>
            <!-- ServiceDefinition 定义了服务的信息 -->
            <bean class="com.appchina.rpc.remote.ServiceDefinition">
                <!-- 可选，用于区分不同实现类 -->
                <property name="serviceName" value="addServiceImpl"></property>
                <!-- 发布服务的接口 -->
                <property name="interfaceClass" value="com.appchina.rpc.test.api.AddService"></property>
                <!-- 发布服务实现类 -->
                <property name="implInstance">
                    <bean class="com.appchina.rpc.test.impl.AddServiceImpl" />
                </property>
            </bean> 
        </array>
    </property> 
</bean>  

<bean class="com.appchina.rpc.thrift.server.ThriftThreadPoolServer">
    <property name="processor" ref="servicePublisher"></property>
    <property name="port" value="9090"></property>
    <property name="minWorkerThreads" value="100"></property>
    <property name="workerThreads" value="500"></property>
    <property name="security" value="true"></property>
    <property name="stopTimeoutVal" value="3000"></property>
    <property name="clientTimeout" value="10000"></property>
    <property name="allowedFromTokens">
        <map>
            <entry key="DONGJIAN" value="DSIksduiKIOYUIOkYIOhIOUIOhjklYUI"></entry>
        </map>
    </property>
</bean> 
### 客户端：
/appchina-rpc-test/src/main/java/main/Client.java

ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("/application-client.xml");
AddService addService = (AddService) ctx.getBean("addService");
addService.add(100);

/appchina-rpc-test/src/main/resources/application-client.xml

<bean id="factoryProvider" class="com.appchina.rpc.thrift.cluster.ThriftClientFactoryProvider">    
    <!-- server列表 -->       
    <property name="hostPorts"> 
        <list>                   
            <value>127.0.0.1:9090</value>                
            <value>127.0.0.1:9191</value>
        </list>
    </property>
    <!-- 请求超时，根据业务设置 -->
    <property name="timeout" value="60000"></property>
    <!-- 连接超时，超过这个时间无法创建连接的server将被设置为暂时无效，恢复时设置为有效   -->
    <property name="connectionTimeout" value="10000"></property>
    <!-- 如果服务端是NIO，需要启用此配置 -->
    <property name="framed" value="false"></property>
    <!-- 安全选项 -->
    <property name="from" value="DONGJIAN"></property>
    <property name="token" value="DSIksduiKIOYUIOkYIOhIOUIOhjklYUI"></property>
</bean>


<!-- 关于集群的相关配置 -->   
<bean id="client" class="com.appchina.rpc.base.cluster.client.DistributeClient">   
    <property name="factoryProvider" ref="factoryProvider"></property>   
    <!-- 负载均衡实现类 -->   
    <property name="loadBalance">   
        <bean class="com.appchina.rpc.base.cluster.RoundrobinLoadBalance"></bean>   
    </property>   
    <!-- 心跳频率，用于检测Server可用性的间隔 -->  
    <property name="heartbeat" value="1000"></property>   
    <!-- 处理心跳的最大线程数，一般1个线程足够 -->   
    <property name="maxHeartbeatThread" value="1"></property>   
    <!-- 连接池耗完直接重试，重试其他池子的次数，因此、maxWait的时间可能叠加 -->   
    <property name="retry" value="3"></property>                 
    <!-- 由于被借走时间不一样，可能导致单个池子不够用，建议这个值大一些，可以通过maxIdle来限制长连接数量 -->                          
    <property name="maxActive" value="300"></property>                  
    <!-- 最大空闲，当池内连接大于maxIdle，每次returnObject都会销毁连接，maxIdle保证了长连接的数量 -->                  
    <property name="maxIdle" value="200"></property>                
    <!-- 最小空闲 -->              
    <property name="minIdle" value="5"></property>                  
    <!-- 等待连接池的时间 -->                
    <property name="maxWait" value="1000"></property>                   
    <!-- 连接使用多久之后被销毁 -->               
    <property name="maxKeepMillis" value="-1"></property>                  
    <!-- 连接使用多少次之后被销毁 -->                  
    <property name="maxSendCount" value="-1"></property>                  
</bean> 

<bean id="addService" class="com.appchina.rpc.thrift.remote.base.ThriftRemoteProxyFactory">  
    <property name="serviceName" value="addService"></property>     
    <property name="proxyInterface" value="com.appchina.rpc.test.api.AddService"></property>
    <property name="client" ref="client"></property>        
</bean>      
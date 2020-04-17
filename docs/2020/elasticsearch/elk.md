### SpringBoot2.x日志收集搭建 ELK(7.6.2)+(RabbitMq3.7.16+Erlang 21.0.1)
项目地址：[sb-elk-rabbitmq](https://github.com/Pamgo/sb-elasticsearch-demo.git)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020041714214396.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70#pic_center)
* [rabbitmq-3.7.6](https://www.rabbitmq.com/) 自行到官网下载安装对应的版本以及对应的Erlang可参考[地址](https://blog.csdn.net/newbie_907486852/article/details/79788471)

* [elasticsearch-7.6.2](https://www.elastic.co/cn/start)(下载解压即可)  [集群搭建选看](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html)
* [elasticsearch-head](https://github.com/mobz/elasticsearch-head)(下载解压放到对应目录下，稍后讲解)

* [kibana-7.6.2](https://www.elastic.co/cn/start)（下载解压即可）

* [logstash-7.6.2](https://www.elastic.co/cn/logstash)（下载解压即可）

  在系统中elk文件夹放置以下文件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200417115127740.png#pic_center)

> 1、配置elasticsearch

打开elasticsearch的目录下elasticsearch.yml文件添加配置

```yaml
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
# 配置elasticsearch数据目录
path.data: D:\dev\devsoft\elk\data
#
# Path to log files:
# 配置elasticsearch日志目录
path.logs: D:\dev\devsoft\elk\logs
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
#network.host: 192.168.0.1
#
# Set a custom port for HTTP:
#
#http.port: 9200
# 以下两个配置问了配置elasticsearch-head防止跨域问题
http.cors.enabled: true
http.cors.allow-origin: "*"
```

在elasticsearch的bin目录下双击`elasticsearch.bat`文件即可启动

启动成功后访问： http://localhost:9200

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200417115148825.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70#pic_center)

> 2、配置elasticsearch-head插件

在elasticsearch-head目录下打开命令窗口执行以下命令（前提安装了node.js/npm命令）

```powershell
> npm install
# 启动elasticsearch-head
> npm run start
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200417115208664.png#pic_center)

访问：http://localhost:9100 即可访问

> 3、配置kibana

打开kibana的配置目录conf下的kibana.yml配置文件，添加配置

```yaml
# Kibana is served by a back end server. This setting specifies the port to use.
server.port: 5601
# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
server.host: "localhost"
# The URLs of the Elasticsearch instances to use for all your queries.
elasticsearch.hosts: ["http://localhost:9200"]
# Specifies locale to be used for all localizable strings, dates and number formats.
# Supported languages are the following: English - en , by default , Chinese - zh-CN .
#i18n.locale: "en"
i18n.locale: "zh-CN"
```

在bin目录下双击启动文件`kibana.bat`启动

访问：http://localhost:5601
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200417115227718.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70#pic_center)

> 4、logstash接入RabbitMQ

官方文档地址：[访问](https://www.elastic.co/guide/en/logstash/7.6/input-plugins.html)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200417115245214.png#pic_center)

**在logstash的bin目录新建配置文件rabbitmq-log.conf 并加入配置**,具体配置可查看上面官方文档地址

```config
input {
    # 输入配置，这里选用Rabbitmq的配置
    rabbitmq {
        # rabbitmq地址
        host => "127.0.0.1"
        # rabibtmq端口
        port => 5672
        # codec为来源数据格式
        codec => "json"
        # rabbitmq中的交换器
        exchange=> "ex_es_logstash"
        # 监听的mq的queue，设置该项可以不用配置key
        queue => "es-log-queue"
        # 监听的路由key
        key => "elk-es-log"
        # queue是否持久化
        durable => true
        # type内容可以自由定义，可作为标识
        type => "es"
    }
}

filter {
    # 过滤，非必须
    # input使用codec为json格式，可以不进行grok正则匹配处理
}

output {
    # 在字符串外的变量使用中括号"[]"包裹，如[type]
    if [type] == "es" {
        elasticsearch {
            # elasticsearch地址
            hosts => ["127.0.0.1:9200"]
            # 根据输入的type类型与爬虫名称构建index名称
            # 字符串内变量使用"%{}"包裹，如%{type}
            index => "es-%{type}_log-%{+YYYY.MM.dd}"
        }
    } else {
        elasticsearch {
            hosts => ["127.0.0.1:9200"]
            index => "es-%{type}-log-%{+YYYY.MM.dd}"
        }
    }
}
```

在bin目录下打开命令窗口执行以下命令

```powershell
> logstash.bat -f rabbitmq-log.conf
```

> 5、项目中日志接入rabbitmq收集

rabbitmq包引入
```xml
<dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-amqp</artifactId>
     <version>2.2.1.RELEASE</version>
 </dependency>
```
在项目中的日志配置文件`logback-spring.xml`中配置以下信息

```xml
<appender name="AMQP" class="org.springframework.amqp.rabbit.logback.AmqpAppender">
        <layout>
            <pattern>
                {
                    "time": "%date{ISO8601}",
                    "thread": "%thread",
                    "level":
                    "%level",
                    "class": "%logger{60}",
                    "message": "%msg"
                }
            </pattern>
        </layout>
        <!--rabbitmq地址-->
        <host>127.0.0.1</host>
        <!--端口-->
        <port>5672</port>
        <!--用户名-->
        <username>admin</username>
        <!--密码-->
        <password>root</password>
        <!--项目名-->
        <applicationId>byterun-es-service</applicationId>
        <!--rabbitmq队列接收的路由key和logstash的rabbitmq-log.conf中的路由key对应-->
        <routingKeyPattern>elk-es-log</routingKeyPattern>
        <declareExchange>true</declareExchange>
        <exchangeType>direct</exchangeType>
        <!--rabbitmq交换器名称，和logstash的rabbitmq-log.conf的交换器对应-->
        <exchangeName>ex_es_logstash</exchangeName>
        <generateId>true</generateId>
        <charset>UTF-8</charset>
        <durable>true</durable>
        <deliveryMode>PERSISTENT</deliveryMode>
    </appender>
   <!--info日志输出到rabbitmq-->
 	<root level="INFO">
        <appender-ref ref="AMQP" />
    </root>
   <!--项目日志debug日志输出到rabbitmq-->
	<logger name="com.xxx" level="DEBUG" additivity="false">
        <appender-ref ref="AMQP" />
    </logger>
```
具体可用配置请查看源码`org.springframework.amqp.rabbit.logback.AmqpAppender`
如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200417164927680.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70)

完整配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!--输出到控制台-->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</Pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>
    <!-- ch.qos.logback.core.rolling.RollingFileAppender 文件日志输出 -->
    <appender name="FILE"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/byterun-es-service.log</file>

        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/byterun-es-service.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
            <maxHistory>10</maxHistory>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <MaxFileSize>30MB</MaxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE-ERROR"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/byterun-es-service.err</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/byterun-es-service.%d{yyyy-MM-dd}-%i.err</fileNamePattern>
            <maxHistory>10</maxHistory>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <MaxFileSize>30MB</MaxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    <appender name="AMQP" class="org.springframework.amqp.rabbit.logback.AmqpAppender">
        <layout>
            <pattern>
                {
                    "time": "%date{ISO8601}",
                    "thread": "%thread",
                    "level":
                    "%level",
                    "class": "%logger{60}",
                    "message": "%msg"
                }
            </pattern>
        </layout>
        <host>127.0.0.1</host>
        <port>5672</port>
        <username>admin</username>
        <password>root</password>
        <applicationId>byterun-es-service</applicationId>
        <routingKeyPattern>elk-es-log</routingKeyPattern>
        <declareExchange>true</declareExchange>
        <exchangeType>direct</exchangeType>
        <exchangeName>ex_es_logstash</exchangeName>
        <generateId>true</generateId>
        <charset>UTF-8</charset>
        <durable>true</durable>
        <deliveryMode>PERSISTENT</deliveryMode>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="FILE-ERROR"/>
        <appender-ref ref="AMQP" />
    </root>

    <logger name="com.es" level="DEBUG" additivity="false">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="FILE-ERROR"/>
        <appender-ref ref="AMQP" />
    </logger>

    <appender name="accessLog" class="ch.qos.logback.core.FileAppender">
        <file>logs/access_log.log</file>
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>

    <appender name="async" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="accessLog"/>
    </appender>

    <logger name="org.springframework.jdbc.core" level="DEBUG" additivity="false">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE"/>
    </logger>

    <appender name="eventTrackLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/byterun-es-service-event-track.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/byterun-es-service-event-track.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
            <maxHistory>10</maxHistory>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <MaxFileSize>30MB</MaxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="service-event-track" level="INFO" additivity="false">
        <appender-ref ref="eventTrackLog"/>
    </logger>

</configuration>
```

启动顺序` elasticsearch=>elasticsearch-head=>kibana=>logstash=>项目`

访问项目即可在kibana中查看到对应的日志
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200417115316822.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIxNTAxNjg=,size_16,color_FFFFFF,t_70#pic_center)
参考文献：

* [ELK实时日志分析平台环境部署--完整记录](https://www.cnblogs.com/kevingrace/p/5919021.html)
* [Elasticsearch 7.x 最详细安装及配置](https://www.jianshu.com/p/cce06b931030)
* [日志管理之自定义Appender](https://blog.csdn.net/xie19900123/article/details/82056732)
* [ELK环境部署](https://www.toutiao.com/i6815826012362768907/)
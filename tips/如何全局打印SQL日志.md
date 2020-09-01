# 如何全局打印SQL日志

工作这么多年了，从没见过哪个项目里面有这个功能，不知道是缺乏想象力还是怎么滴。
用 `MyBatis` 或 `JPA(Hibernate)` 都是开启他们自己的 `SQL` 打印。
假如中间有用 `jdbc` 直接查询的话，基本上就不走他们的打印了，毕竟也不属于 `MyBatis` 或 `JPA` 的管辖范围。
那怎么样才能全局打印 SQL 日志呢？

## 如何下手

从需求来看，我们要全局拦截，首先就要想，谁能做到全局的查询都经它手呢？  

那好像也就 `jdbc` 驱动能行了。

## 验证想法

`jdbc` 驱动这个，仔细想想它得分为两种，一种是规范，另一种就是具体实现。如果规范里面有相关定义可以让我们配置一下就能记录下所有的 `SQL` 记录，那便是极好的。
那如果规范里面没有，我们就只能把想法寄托于 `MySQL JDBC Driver` 了。

### JDBC 规范

搜 `jdbc doc` 基本上就能找到, 简单翻翻你就能得出结论: `没有`。

### MySQL JDBC Driver

[MySQL JDBC Doc](https://dev.mysql.com/doc/connectors/en/connector-j-reference-configuration-properties.html) 点开搜 `logger` 即能找到 `Debugging/Profiling` 章节。  

说明这个 `MySQL` 的驱动集成了相关的功能，用于 `debug` 或 `profiling`，完美。

下面来看看支持的参数:

* profileSQL  
    Trace queries and their execution/fetch times to the configured 'profilerEventHandler'
    Default: false
* maxQuerySizeToLog  
    Controls the maximum length of the part of a query that will get logged when profiling or tracing
    Default: 2048
* profilerEventHandler  
    Name of a class that implements the interface com.mysql.cj.log.ProfilerEventHandler that will be used to handle profiling/tracing events.  
    Default: com.mysql.cj.log.LoggingProfilerEventHandler
* logger  
    默认是  com.mysql.cj.log.StandardLogger

说人话：  
这个驱动支持上面这4个参数配置，其中都有默认值，整体默认表现就是： `没有表现`。  
想要有得开一个参数:

```plain
profileSQL=true
```

这是最基本的需求了，这样配置以后就可以实现打印了。当然，打印的内容啥的都是默认的格式，想要改就得继续去修改这四个参数中的另外3个。  

> PS： jdbc 默认用的是自己实现的 log 不太好控制，最好还是自己重新实现一遍用 slf4j 处理一下。

## 样例

```kotlin
package com.xxx.core.jpa

import com.mysql.cj.Query
import com.mysql.cj.Session
import com.mysql.cj.log.Log
import com.mysql.cj.log.ProfilerEvent
import com.mysql.cj.log.ProfilerEventHandler
import com.mysql.cj.protocol.Resultset
import org.slf4j.LoggerFactory

/**
 * 此类应用于 jdbc url 中给 MySQL jdbc 定制日志打印用
 * https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-reference-configuration-properties.html
 */
class MyProfilerEventHandler : ProfilerEventHandler {

    private val logger = LoggerFactory.getLogger(SogProfilerEventHandler::class.java)

    override fun destroy() {
    }

    private val eventTypeName = arrayOf(
            "USAGE",
            "OBJECT_CREATION",
            "PREPARE",
            "QUERY",
            "EXECUTE",
            "FETCH",
            "SLOW_QUERY"
    )

    override fun processEvent(eventType: Byte,
                              session: Session?,
                              query: Query?,
                              resultSet: Resultset?,
                              eventDuration: Long,
                              eventCreationPoint: Throwable?,
                              message: String?) {
        if (logger.isDebugEnabled) {
            val eventName = eventTypeName.getOrNull(eventType.toInt())
            logger.debug("event:$eventName,id:${query?.id},duration:$eventDuration,message:$message")
        }
    }

    override fun init(log: Log?) {

    }

    override fun consumeEvent(evt: ProfilerEvent?) {

    }
}
```

然后在 `jdbc` 配置里面添加参数

```yaml
datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/xxx?characterEncoding=UTF-8&useUnicode=true&useSSL=false&serverTimeZone=Asia/Shanghai&profileSQL=true&profilerEventHandler=com.xxx.core.jpa.MyProfilerEventHandler
```

这样项目里面就有了日志了。  

# 科普一下ELK进行日志收集

本文是我学习elk的一点知识总结，源于对网上一些资料的提取总结。旨在科普一下elk技术栈应用于日志收集与搜索的一些要点。

### ELK日志收集与搜索
ELK 是三款软件的组合。是一整套完整的解决方案。分别是由 Logstash（收集+分析）、ElasticSearch（搜索+存储）、Kibana（可视化展示）三款软件。
ELK主要是为了在海量的日志系统里面实现分布式日志数据集中式管理和查询，便于监控以及排查故障。

- Logstash ：数据收集处理引擎，具备实时管道处理能力。支持动态的从各种数据源搜集数据，并对数据进行过滤、分析、丰富、统一格式等操作，然后存储以供后续使用。
- ElasticSearch：分布式搜索引擎。具有高可伸缩、高可靠、近实时等特点。可以用于全文检索、结构化检索和分析，是目前业界使用最广的开源搜索引擎之一。
- Kibana ：可视化平台。能够搜索、展示存储在 Elasticsearch 中的索引数据。使用它可以很方便的用图表、表格、地图展示和分析数据。

elk日志收集与搜索平台基本架构如下：

![elk架构1.png](http://note.youdao.com/yws/res/39196/WEBRESOURCE854407e3aff45a9e0e9cbdbcf12504aa)

- 按照上面这种架构，我们需要在每一台应用服务器上安装部署 Logstash，来采集当前服务器上的日志信息。因为 Logstash 是基于 java 编写的，功能虽然强大，但是它依赖于 java 环境，在日志量很大的情况下会遇到资源占用高的问题，性能问题要考虑。
- 为了解决海量日志的场景下 Logstash 的性能问题，通常会引入了 Beats 组件。目前es官网支持了众多的Beats组件，日志采集中最常用的是Filebeat。
- 在高并发的情况下，由于日志传输峰值比较大，当 Logstash 接收数据的能力超过 Elasticsearch 集群处理能力的时候，就会导致 Elasticsearch 集群丢失数据。
- 为了解决高并发下es丢数据的问题，通常会引入消息队列来做缓冲，通过队列能起到削峰填谷的作用。业界常用于日志采集的消息队列有redis、kafka 和 rabbitmq。

技术演进之后的架构通常是下面这样的：

![elk架构2.png](http://note.youdao.com/yws/res/39227/WEBRESOURCEd3692f01e6ac8a58f5d0c09b26aea7dd)

- Filebeat 是基于 logstash-forwarder 的源码改造而成，用 Golang 编写，无需依赖 Java 环境，效率高，占用内存和 CPU 比较少，具备可靠性和低延迟，非常适合作为 Agent 跑在应用服务器上。
- Grafana 是一个跨平台的开源的度量分析和可视化工具，采用Golang 编写，可以对ELK的结果数据做报表展示、统计监控和业务级别告警。


### Logstash篇
为什么先说一说 logstash？因为我在之前部署整套ELK日志收集搜索系统的时候，遇到最多的问题都是在 logstash 环节，第二个就是我觉得 logstash 需要手动配置的东西也是最多的。
#### 1. Logstash概念

Logstash 是一款开源的日志收集、分析、过滤和传输工具。简单来说，logstash作为数据源与数据存储分析工具之间的桥梁，结合ElasticSearch以及Kibana，能够极大方便数据的处理与分析。

Logstash 支持众多的插件，通过200多个插件，可以接受几乎各种各样的数据。包括日志、网络请求、关系型数据库、传感器或物联网等等。

#### 2. Logstash工作流程

Logstash的数据处理过程中主要涉及的组件包括：Input,Filter,Output 三部分，另外在Input和Output中可以使用Codec对数据格式进行处理。

这四个部分均以插件形式存在，用户通过定义pipeline配置文件，设置需要使用的input,filter,output,codec插件，以实现特定的数据采集，数据处理，数据输出等功能 。

![logstash工作流程1.png](http://note.youdao.com/yws/res/39367/WEBRESOURCEa98a50982d13fbc97ae654a7d8eaf71c)

- Input：用于从数据源获取数据，常见的插件如stdin, file, syslog, redis, beats 等
- Filter：用于处理数据如格式转换，数据派生等，常见的插件如grok, mutate, drop, clone, geoip等
- Output：用于数据输出，常见的插件如elastcisearch，file, graphite, statsd等
- Codec：Codecs(编码插件)不是一个单独的流程，而是在输入和输出等插件中用于数据转换的模块，用于对数据进行编码处理，常见的插件如json，multiline。
- Logstash不只是一个input | filter | output 的数据流，而是一个 input | decode | filter | encode | output 的数据流！codec 就是用来 decode、encode 事件的。

#### 3. Logstash插件

logstash支持的插件非常丰富，功能很强大。下面会列举几个常用的插件进行分析和实践：

##### 3.1 stdin标准输入

stdin 是 logstash 里最简单和基础的插件了。 input { stdin { } }表示从控制台输入。

配置文件logstash.conf  内容如下：
```
input {
  stdin {
    codec => line
  }
}

filter {}

output {
  stdout {
    codec => rubydebug
  }
}
```

输入输出示例解析：
- 输入

![logstash input1.png](http://note.youdao.com/yws/res/39408/WEBRESOURCEe14610aaf6831069a52718115917dbb9)

- 输出

![logstash output1.png](http://note.youdao.com/yws/res/39417/WEBRESOURCEf41c923fb1f8b83b0c026b9a57c6f603)


logstash标准输入到标准输出(收集终端实时日志并实时展示)：
- 输入一行普通文本：

![std2std.png](http://note.youdao.com/yws/res/39497/WEBRESOURCEaf45f4fe8639045f4c171611c7dafc48)

- 输入一行json数据：

![std_json.png](http://note.youdao.com/yws/res/39489/WEBRESOURCEd6616ef6191a86bef87f69b06d355ae1)

##### 3.2 Dissect插件

dissect 基于分隔符原理解析数据，解决grok解析时消耗过多 cpu 资源的问题。dissect 语法简单，能处理的场景比较有限。它只能处理格式相似，且有分隔符的字符串。

它的语法如下：
- 1、%{}里面是字段
- 2、两个%{}之间是分隔符

日志样例如下：
```shell
[2020-06-04 19:21:13] [INFO ] [GOID:1] (main counter.go main:62) 第266次，拒绝访问!!!
```

配置文件logstash.conf    内容如下：
```
input {
  stdin {
    codec => line
  }
}

filter {
  dissect {
        mapping => {
        	"message" => "[%{timestamp}] [%{info}] [GOID:%{goid}] (%{path}) %{msg}"
        }
  }
}

output {
  stdout {
    codec => rubydebug
  }
}
```

- 可以取到自定义的一些字段的值，在终端上的效果如下：

![desect1.png](http://note.youdao.com/yws/res/39467/WEBRESOURCE542e454dc0481768e5f063f3a9367a69)



### 未完待续 ......





# 安装ELK进行日志收集的踩坑实践

### ELK全家桶
- **elasticsearch集群**
- **logstash日志采集组件**
- **kibana日志可视化组件**

### elasticsearch集群篇
#### 1. 在安装es集群之前，首先需要安装java环境，因为es/logstash/kibana都是用java编写的，依赖jvm运行环境：

去oracle官网下载jdk1.8的包，我放在了~/java-home目录下
```golang
steve@stevedeMac-mini  ~  cd java-home
steve@stevedeMac-mini  ~/java-home  ls
jdk-8u251-macosx-x64.dmg
steve@stevedeMac-mini  ~/java-home 
steve@stevedeMac-mini  ~/java-home 
```

安装好之后，默认路径为：
/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home

需要设置一下环境变量：
终端输入：sudo vim /etc/profile
编辑profile信息，在后面加入以下信息，保存退出：
```golang
JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home"
export JAVA_HOME
CLASS_PATH="$JAVA_HOME/lib"
PATH=".$PATH:$JAVA_HOME/bin"
```

然后需要使之生效即可！


#### 2. java环境配置好之后，安装es集群，规划3个es节点(1主2从)
##### 2.1 本地先创建es1.yml  es2.yml  es3.yml  这三个文件：
```
steve@stevedeMac-mini  ~/docker-repo/es/config  ls
es1.yml es2.yml es3.yml
steve@stevedeMac-mini  ~/docker-repo/es/config 
steve@stevedeMac-mini  ~/docker-repo/es/config 
```

es1.yml  内容如下：
```golang
#集群唯一名称，所有节点都一样
cluster.name: es-cluster

#节点名称
node.name: es-node1

#设置可以访问的ip，设置0.0.0.0全部通过
network.host: 0.0.0.0

#设置其他节点与该节点交互的ip地址
network.publish_host: 192.168.31.141

#设置对外服务的http端口
http.port: 9200

#设置节点之间交互的tcp端口
transport.tcp.port: 9300

#是否支持跨域
http.cors.enabled: true

#表示支持所有域名
http.cors.allow-origin: "*"

#配置该节点有资格被选举为主节点（候选主节点），为防止脑裂需要配置奇数个候选主节点
node.master: true

#配置该节点为数据节点，保存数据
node.data: true

#集群中各个节点的ip地址
discovery.zen.ping.unicast.hosts: ["192.168.31.141:9300","192.168.31.141:9301","192.168.31.141:9302"]

#设置最少可工作的候选主节点个数，为防止脑裂需要设置为：半数+1
discovery.zen.minimum_master_nodes: 2
```

es2.yml  内容如下：
```golang
cluster.name: es-cluster
node.name: es-node2
network.host: 0.0.0.0
network.publish_host: 192.168.31.141
http.port: 9201
transport.tcp.port: 9301
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: true
node.data: true
discovery.zen.ping.unicast.hosts: ["192.168.31.141:9300","192.168.31.141:9301","192.168.31.141:9302"]
discovery.zen.minimum_master_nodes: 2
```

es3.yml  内容如下：
```golang
cluster.name: es-cluster
node.name: es-node3
network.host: 0.0.0.0
network.publish_host: 192.168.31.141
http.port: 9202
transport.tcp.port: 9302
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: true
node.data: true
discovery.zen.ping.unicast.hosts: ["192.168.31.141:9300","192.168.31.141:9301","192.168.31.141:9302"]
discovery.zen.minimum_master_nodes: 2
```

##### 2.2 拉取es镜像：
```
docker pull elasticsearch:6.7.1
```

##### 2.3 运行ES01  ES02  ES03这三个容器，启动集群：
```
docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9200:9200 -p 9300:9300 -v ~/docker-repo/es/config/es1.yml:/usr/share/elasticsearch/config/elasticsearch.yml --name ES01 e2667f5db289

docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9201:9201 -p 9301:9301 -v ~/docker-repo/es/config/es2.yml:/usr/share/elasticsearch/config/elasticsearch.yml --name ES02 e2667f5db289

docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9202:9202 -p 9302:9302 -v ~/docker-repo/es/config/es3.yml:/usr/share/elasticsearch/config/elasticsearch.yml --name ES03 e2667f5db289
```

OK！通过docker ps可以看到，上面的3个es节点都运行起来了。
通过web页面可以访问到集群：

![1.png](https://github.com/guobingithub/docs-repo/blob/master/images/elk/ELK%E8%BF%9B%E8%A1%8C%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86%E7%9A%84%E9%87%87%E5%9D%91%E5%AE%9E%E8%B7%B5/1.png?raw=true)


或者查看集群节点：

![2.png](https://github.com/guobingithub/docs-repo/blob/master/images/elk/ELK%E8%BF%9B%E8%A1%8C%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86%E7%9A%84%E9%87%87%E5%9D%91%E5%AE%9E%E8%B7%B5/2.png)


集群中有3台节点，es-node1为主节点，es-node2  es-node3为从节点。


#### 3. 接下来，可以安装一个es的可视化Header插件
```
# 拉取镜像
> 1、docker pull mobz/elasticsearch-head:5

# 运行容器
> 2、docker run -d --name es_admin -p 9100:9100 mobz/elasticsearch-head:5

# 等待启动成功
```
接下来，就可以通过es-head插件查看es集群的状态信息：

![3.png](https://github.com/guobingithub/docs-repo/blob/master/images/elk/ELK%E8%BF%9B%E8%A1%8C%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86%E7%9A%84%E9%87%87%E5%9D%91%E5%AE%9E%E8%B7%B5/3.png)


### logstash篇
#### 1. 安装好es集群之后，安装logstash组件
##### 1.1 本地先安装并启动logstash
```
# 拉取镜像
> 1、docker pull logstash:6.7.1

# 运行容器
> 2、docker run -d --name logstash logstash:6.7.1

# 等待启动成功
```
实际上，由于没有配置任何的配置文件，以及可能存在的x-pack安全验证等问题，上面启动logstash容器的时候是会报错的。此时，先不用管先启动再说。

##### 1.2 设置配置文件信息

> 为了方便更改logstash配置文件，以及 piepline 下面定义的 logstash.conf，我把这两个文件夹复制到宿主机的挂载目录。

##### 1.2.1 即把 /usr/share/logstash/config 以及 /usr/share/logstash/pipeline 文件夹复制到宿主机：
```shell
steve@stevedeMac-mini  ~/docker-repo/logstash 
steve@stevedeMac-mini  ~/docker-repo/logstash  docker cp logstash:/usr/share/logstash/config ~/docker-repo/logstash
steve@stevedeMac-mini  ~/docker-repo/logstash  docker cp logstash:/usr/share/logstash/pipeline ~/docker-repo/logstash
steve@stevedeMac-mini  ~/docker-repo/logstash 
steve@stevedeMac-mini  ~/docker-repo/logstash  ls
config   pipeline
steve@stevedeMac-mini  ~/docker-repo/logstash 
steve@stevedeMac-mini  ~/docker-repo/logstash 
```

##### 1.2.2 在设置配置文件之前，先创建一个自己的日志文件目录，并准备一份日志文件，以便于后面进行测试：
> 我创建的是mylog文件夹，并准备了一份mylog.log日志文件

```shell
steve@stevedeMac-mini  ~/docker-repo/logstash 
steve@stevedeMac-mini  ~/docker-repo/logstash  ls
config   mylog    pipeline
steve@stevedeMac-mini  ~/docker-repo/logstash 
steve@stevedeMac-mini  ~/docker-repo/logstash  ls mylog
mylog.log
steve@stevedeMac-mini  ~/docker-repo/logstash 
steve@stevedeMac-mini  ~/docker-repo/logstash 
```

##### 1.2.3 OK！接下来设置几个关键的配置文件：
```shell
steve@stevedeMac-mini  ~/docker-repo/logstash 
steve@stevedeMac-mini  ~/docker-repo/logstash  cd config
steve@stevedeMac-mini  ~/docker-repo/logstash/config  ls
jvm.options          logstash-sample.conf logstash.yml         pipelines.yml
log4j2.properties    logstash.conf        startup.options
steve@stevedeMac-mini  ~/docker-repo/logstash/config 
```

logstash.yml  内容如下：
```golang
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "192.168.31.141:9200","192.168.31.141:9201","192.168.31.141:9202" ]
xpack.monitoring.elasticsearch.username: "elastic"
xpack.monitoring.elasticsearch.password: "changeme"
xpack.monitoring.enabled: true
```

logstash.conf  内容如下：
```golang
input {
  file {
    path => "/usr/share/logstash/mylog/mylog.log"
    codec => "json"
    type => "mylog"
    start_position => "beginning"
  }
}

filter {}

output {
  elasticsearch {
    hosts => ["192.168.31.141:9200","192.168.31.141:9201","192.168.31.141:9202"]
    user => "elastic"
    password => "changeme"
    index => "logstash-es-%{+YYYY.MM.dd}"
  }
}
```

pipelines.yml  内容如下：
```golang
- pipeline.id: main
  path.config: "/usr/share/logstash/config/logstash.conf"
  pipeline.workers: 3
```


##### 1.3 准备好日志文件，并设置好配置文件之后，即可重新启动logstash服务
```docker
docker run --restart=always  -m 1000M \
-it -d -p 5044:5044 -p 9600:9600 --name logstash \
-v ~/docker-repo/logstash/mylog:/usr/share/logstash/mylog \
-v ~/docker-repo/logstash/config:/usr/share/logstash/config \
-v ~/docker-repo/logstash/pipeline:/usr/share/logstash/pipeline \
--privileged=true \
logstash:6.7.1
```

##### 1.4 重启logstash之后，可以查看日志确保没有报错信息
```
docker logs -f logstash
```

#### 2. 安装logstash可能遇到的问题
##### 2.1 镜像版本问题
- 起初，我安装的es版本是7.7.0，然后logstash版本是6.7.1版本
- 启动logstash的时候一直会报错，看日志跟版本有关
- 网上有看到资料显示es logstash kibana他们的版本最好一致
- 于是，重新拉取了6.7.1版本的es镜像，并且重启es集群，重启logstash服务

##### 2.2 x-pack安全问题
由于ES6.X以上的版本自带了x-pack安全认证插件，因此6.7.1版本在安装的时候也可能会遇到x-pack问题，通常需要修改密码：
```golang
# es
curl -XPUT -u elastic '192.168.31.141:9200/_xpack/security/user/elastic/_password' -H "Content-Type: application/json" -d '{ "password" : "123456"}'

# logstash
curl -XPUT -u elastic '192.168.31.141:9200/_xpack/security/user/logstash_system/_password' -H "Content-Type: application/json" -d '{ "password" : "123456"}'

# kibana
curl -XPUT -u elastic '192.168.31.141:9200/_xpack/security/user/kibana/_password' -H "Content-Type: application/json" -d '{ "password" : "123456"}'
```

##### 2.3 license问题
查看日志，可能会遇到的如下license问题：
```
{
    "error":{
        "root_cause":[
            {
                "type":"security_exception",
                "reason":"current license is non-compliant for [security]",
                "license.expired.feature":"security"
            }],
        "type":"security_exception",
        "reason":"current license is non-compliant for [security]",
        "license.expired.feature":"security"
    },
    "status":403
}
```

那么，你可能需要去ES官网申请对应版本的许可证

- 注册elasticsearch账号,注册地址 https://register.elastic.co/
- 根据你注册填写的邮箱，会收到一封邮件，进入邮件获取许可证下载地址
- 下载对应版本的许可证，下载好上传到ES服务器，并根据手册执行安装命令
```
steve@stevedeMac-mini  ~/docker-repo/es 
steve@stevedeMac-mini  ~/docker-repo/es  ls
config                                                bin-guo-19f181e8-c62c-4ccf-97c3-a1674e6f6fd2-v5.json
steve@stevedeMac-mini  ~/docker-repo/es 
steve@stevedeMac-mini  ~/docker-repo/es  curl -XPUT -u elastic 'http://192.168.31.141:9200/_xpack/license?acknowledge=true' -H "Content-Type: application/json" -d @bin-guo-19f181e8-c62c-4ccf-97c3-a1674e6f6fd2-v5.json
```

然后，可以通过curl查看license信息：
```golang
steve@stevedeMac-mini  ~/docker-repo/logstash  curl -XGET -u elastic 'http://192.168.31.141:9200/_license'
Enter host password for user 'elastic':
{
  "license" : {
    "status" : "active",
    "uid" : "196aa5ce-24f3-4d01-b0f8-f19c3337a54c",
    "type" : "basic",
    "issue_date" : "2020-06-02T07:45:49.972Z",
    "issue_date_in_millis" : 1591083949972,
    "max_nodes" : 1000,
    "issued_to" : "es-cluster",
    "issuer" : "elasticsearch",
    "start_date_in_millis" : -1
  }
}
```

或者web页面查看：

![4.png](https://github.com/guobingithub/docs-repo/blob/master/images/elk/ELK%E8%BF%9B%E8%A1%8C%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86%E7%9A%84%E9%87%87%E5%9D%91%E5%AE%9E%E8%B7%B5/4.png)

##### 2.4 挂载文件问题

- 挂载文件问题，主要是说在运行容器之前需要先配置好相关配置文件，并在运行时将他们挂载到docker容器当中
- 比如logstash，我先准备好了config pipeline mylog这三个文件夹，有对应的配置文件和测试用的日志文件
- 然后在docker run启动logstash时，通过-v将这些目录进行挂载映射
```docker
docker run --restart=always  -m 1000M \
-it -d -p 5044:5044 -p 9600:9600 --name logstash \
-v ~/docker-repo/logstash/mylog:/usr/share/logstash/mylog \
-v ~/docker-repo/logstash/config:/usr/share/logstash/config \
-v ~/docker-repo/logstash/pipeline:/usr/share/logstash/pipeline \
--privileged=true \
logstash:6.7.1
```

##### 2.5 配置文件问题
- 这里主要是说logstash的配置文件，logstash的配置文件问题隐藏的比较深
- 我在配置好logstash.yml和logstash.conf之后，满怀期待的等待logstash采集本地的mylog.log日志到es集群当中，但不幸的是发现es中一直没有期望的日志数据
- 后来怀疑是不是受到默认的pipelines.yml文件的影响，于是打开该文件
```
steve@stevedeMac-mini  ~/docker-repo/logstash/config 
steve@stevedeMac-mini  ~/docker-repo/logstash/config  cat pipelines.yml
# This file is where you define your pipelines. You can define multiple.
# For more information on multiple pipelines, see the documentation:
#   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

- pipeline.id: main
  path.config: "/usr/share/logstash/pipeline"
```
- OK！于是打开/usr/share/logstash/pipeline，查看
```
steve@stevedeMac-mini  ~/docker-repo/logstash/config 
steve@stevedeMac-mini  ~/docker-repo/logstash/config  cd ..
steve@stevedeMac-mini  ~/docker-repo/logstash  ls
config   mylog    pipeline
steve@stevedeMac-mini  ~/docker-repo/logstash  cd pipeline
steve@stevedeMac-mini  ~/docker-repo/logstash/pipeline  ls
logstash.conf
steve@stevedeMac-mini  ~/docker-repo/logstash/pipeline 
steve@stevedeMac-mini  ~/docker-repo/logstash/pipeline  cat logstash.conf
input {
  beats {
    port => 5044
  }
}

output {
  stdout {
    codec => rubydebug
  }
}

steve@stevedeMac-mini  ~/docker-repo/logstash/pipeline 
steve@stevedeMac-mini  ~/docker-repo/logstash/pipeline 
```
- OMG！原来logstash默认的配置文件中输入的是beats插件的数据，并没有读取我预先准备好的mylog日志文件。于是修改pipelines.yml文件
```
steve@stevedeMac-mini  ~/docker-repo/logstash/config 
steve@stevedeMac-mini  ~/docker-repo/logstash/config  cat pipelines.yml
- pipeline.id: main
  path.config: "/usr/share/logstash/config/logstash.conf"
  pipeline.workers: 3
steve@stevedeMac-mini  ~/docker-repo/logstash/config 
steve@stevedeMac-mini  ~/docker-repo/logstash/config 
```

- OK！修改好配置文件之后重启logstash服务，惊喜的看到了es中有了预先准备好的日志数据


##### 2.6 内存占用问题
- 内存占用问题，隐藏的也比较深。
- 其实，就是在我本地docker中启动好es集群之后，再启动logstash和kibana的时候，总是会出现莫名其妙的退出重启
- 由于认识不足，刚开始一直以为是logstash和kibana的配置文件没有配置好，怀疑是配置文件或路径的问题
- 不停尝试修改和检查配置一直未果，偶然将启动时的内存限制小一点发现不会莫名的退出了，但是同时启动logstash和kibana又会出现启动不来的现象
- 于是果断怀疑是内存不够的问题。将本地docker的内存从默认的2G调整到3G，重启三个服务果然没有再报错或者退出重启啦！


### kibana篇
#### 1. 安装好es集群和logstash之后，安装kibana组件
##### 1.1 本地先安装并启动kibana
```
# 拉取镜像
> 1、docker pull kibana:6.7.1

# 运行容器
> 2、docker run -it -d -p 5601:5601 --name kibana kibana:6.7.1

# 等待启动成功
```
由于没有设置配置文件，上面启动kibana容器的时候是会报错的。同样的，先不用管先启动再说。

##### 1.2 设置配置文件信息

> 为了方便更改kibana配置文件，我把kibana的config文件夹复制到宿主机的挂载目录。

##### 1.2.1 即把 /usr/share/kibana/config 文件夹复制到宿主机：
```shell
steve@stevedeMac-mini  ~/docker-repo/kibana 
steve@stevedeMac-mini  ~/docker-repo/kibana  docker cp kibana:/usr/share/kibana/config ~/docker-repo/kibana/config
steve@stevedeMac-mini  ~/docker-repo/kibana 
steve@stevedeMac-mini  ~/docker-repo/kibana  ls
config
steve@stevedeMac-mini  ~/docker-repo/kibana  cd config
steve@stevedeMac-mini  ~/docker-repo/kibana/config  ls
kibana.yml
steve@stevedeMac-mini  ~/docker-repo/kibana/config 
steve@stevedeMac-mini  ~/docker-repo/kibana/config 
```

##### 1.2.2 OK！接下来设置kibana.yml配置文件：

kibana.yml  内容如下：
```golang
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://192.168.31.141:9200","http://192.168.31.141:9201","http://192.168.31.141:9202" ]
elasticsearch.username: "elastic"
elasticsearch.password: "changeme"
xpack.monitoring.ui.container.elasticsearch.enabled: true
```

##### 1.3 设置好配置文件之后，即可重新启动kibana服务
```docker
docker run --restart=always \
-it -d -p 5601:5601 --name kibana \
-v ~/docker-repo/kibana/config:/usr/share/kibana/config \
kibana:6.7.1
```

##### 1.4 重启kibana之后，可以查看日志确保没有报错信息
```
docker logs -f kibana
```

##### 1.5 登录web页面，查看kibana可视化效果

索引

![5.png](https://github.com/guobingithub/docs-repo/blob/master/images/elk/ELK%E8%BF%9B%E8%A1%8C%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86%E7%9A%84%E9%87%87%E5%9D%91%E5%AE%9E%E8%B7%B5/5.png)


日志

![6.png](https://github.com/guobingithub/docs-repo/blob/master/images/elk/ELK%E8%BF%9B%E8%A1%8C%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86%E7%9A%84%E9%87%87%E5%9D%91%E5%AE%9E%E8%B7%B5/6.png)


OK！预期的索引和日志都展示出来了，基于elk日志的收集和展示初步搭建成功！



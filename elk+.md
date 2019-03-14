# ES-Dockerfile

FROM java:8-jre
MAINTAINER "30040852@qq.com"

RUN groupadd -r elasticsearch && useradd -r -g elasticsearch elasticsearch

ENV GOSU_VERSION 1.10

RUN set -ex \
    && mkdir -p /usr/local/java \
    && mkdir -p /usr/local/data \
    && mkdir -p /usr/local/data/elasticsearch \
    && mkdir -p /usr/local/data/elasticsearch/data \
    && mkdir -p /usr/local/data/elasticsearch/logs

RUN set -x \
    && apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
    && wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
    && wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
    && export GNUPGHOME="$(mktemp -d)" \
    && gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
    && gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
    && rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
    && chmod +x /usr/local/bin/gosu \
    && gosu nobody true \
    && apt-get purge -y --auto-remove ca-certificates wget

COPY /soft/elasticsearch-6.1.3.tar.gz /usr/local/elasticsearch-6.1.3.tar.gz
COPY /soft/jdk-8u181-linux-x64.tar.gz /usr/local/jdk-8u181-linux-x64.tar.gz

RUN tar xzf /usr/local/jdk-8u181-linux-x64.tar.gz -C /usr/local/java/ && rm -rf /usr/local/jdk-8u181-linux-x64.tar.gz
RUN tar xzf /usr/local/elasticsearch-6.1.3.tar.gz -C /usr/local/ && rm -rf /usr/local/elasticsearch-6.1.3.tar.gz

ENV JAVA_HOME /usr/local/java/jdk1.8.0_181
ENV CLASSPATH .:%JAVA_HOME%\lib:%JAVA_HOME%\lib\dt.jar:%JAVA_HOME%\lib\tools.jar
ENV PATH /usr/local/elasticsearch-6.1.3/bin:%JAVA_HOME%\bin:%JAVA_HOME%\jre\bin:$PATH

WORKDIR /usr/local/elasticsearch-6.1.3

RUN set -ex \
    && for path in \
        ./data \
        ./logs \
        ./config \
        ./config/scripts \
    ; do \
        mkdir -p "$path"; \
        chown -R elasticsearch:elasticsearch "$path"; \
        chown -R elasticsearch:elasticsearch /usr/local/data; \
        chown -R elasticsearch:elasticsearch /usr/local/elasticsearch-6.1.3; \
        chown -R elasticsearch:elasticsearch /usr/local/data/elasticsearch/data; \
        chown -R elasticsearch:elasticsearch /usr/local/data/elasticsearch/logs; \
    done

COPY /config/elasticsearch.yml ./config/elasticsearch.yml

VOLUME /usr/local/elasticsearch-6.1.3/data

COPY docker-entrypoint.sh /

RUN chmod 777 /docker-entrypoint.sh

EXPOSE 9200 9300

ENTRYPOINT ["/docker-entrypoint.sh"]

1. CMD ["elasticsearch"	]



# ES-compose搭建

1、下载elasticsearch镜像

```
docker pull elasticsearch:6.5.4
```

2、安装docker-compose

```
curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

3、更改执行权限

```
 chmod +x /usr/local/bin/docker-compose
```

4、查看是否安装成功

```
docker-compose --version
```

5、elasticsearch不能用root用户执行，创建新用户

```
useradd  elastic
```

6 、创建es容器PATH_data在宿主机的映射文件，并添加权限,修改属主/组

```
mkdir -p /data/es-date
mkdir /root/elastic
chmod -R 777 /data
chown -R elasticsearch.elasticsearch /data
```

7、配置docker-compose配置文件和elasticsearch.yml配置文件

  vim /root/elastic/docker-compose.yml

```
version: '2.2'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.5.4
    container_name: es1
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /alidata2/elasticsearch/data/es-data/:/usr/share/elasticsearch/data
      - ./elasticsearch.yml:/usr/local/share/config/elasticsearch.yml
      - /alidata2/elasticsearch/data/logs/:/usr/local/share/logs
    ports:
      - 9208:9200
      - 9308:9300
    networks:
      - esnet
#  elasticsearch2:
#    image: docker.elastic.co/elasticsearch/elasticsearch:6.5.4
#    container_name: elasticsearch2
#    environment:
#      - cluster.name=docker-cluster
#      - bootstrap.memory_lock=true
#      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
#      - "discovery.zen.ping.unicast.hosts=elasticsearch"
#    ulimits:
#      memlock:
#        soft: -1
#        hard: -1
#    volumes:
#      - esdata2:/usr/share/elasticsearch/data
#    networks:
#      - esnet
#
volumes:
  esdata1:
    driver: local
#  esdata2:
#    driver: local

networks:
  esnet:

```

  vim elasticsearch.yml

```
cluster.name: cluster-elastic
node.name: elk-node1
path.data: /data/es-data
path.logs: /var/log/elasticsearch/
bootstrap.mlockall: true
network.host: _global_
http.port: 9208
http.cors.enabled: true
http.cors.allow-origin: "*"
```

8.1 、启动docker-compose 

```
pwd
/root/elastic
docker-compose up -d
```

8.2  启动docker-compose up报错，docker版本太低，和docker-compose不兼容

```
ERROR: The Docker Engine version is less than the minimum required by Compose. Your current project requires a Docker Engine of version 1.13.0 or greater.
```

8.3 升级docker版本



9.1  进入浏览器所设端口进行验证

公网ip:端口号

```
{
  "name" : "g8vqc0_",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "caA6w3YZRBedwJC_V7f4kw",
  "version" : {
    "number" : "6.5.4",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "d2ef93d",
    "build_date" : "2018-12-17T21:17:40.758843Z",
    "build_snapshot" : false,
    "lucene_version" : "7.5.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

9.2  curl http://内网ip:端口号

```
{
  "name" : "g8vqc0_",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "caA6w3YZRBedwJC_V7f4kw",
  "version" : {
    "number" : "6.5.4",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "d2ef93d",
    "build_date" : "2018-12-17T21:17:40.758843Z",
    "build_snapshot" : false,
    "lucene_version" : "7.5.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

9.3  集群健康值

```
curl http://127.0.0.1:9200/_cat/health
1472225929 15:38:49 docker-cluster green 2 2 4 2 0 0 0 0 - 100.0%
```




# metricbeat搭建

1、下载metricbeat镜像

```
docker pull docker.elastic.co/beats/metricbeat:6.5.4
```

2、下载配置文件

```
curl -L -O https://raw.githubusercontent.com/elastic/beats/6.5/deploy/docker/metricbeat.docker.yml
```

3、创建容器

```
docker run -d \
  --name=metricbeat \
  --user=root \
  --volume="$(pwd)/metricbeat.docker.yml:/usr/share/metricbeat/metricbeat.yml:ro" \
  --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
  --volume="/sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro" \
  --volume="/proc:/hostfs/proc:ro" \
  --volume="/:/hostfs:ro" \
  docker.elastic.co/beats/metricbeat:6.5.4 metricbeat -e \
  -E output.elasticsearch.hosts=["elasticsearch:9200"]
```

metricbeat支持的model有:  Apache,couchbase,Docker,HAProxy,kafka,MongoDB,MySQL,Nginx,PostgreSQL,Prometheus,Redis,System,ZooKeeper  



elasticsearch.yml配置



# elasticsearch报错

## 1、文件属主/组不应该是root

```
Caused by: java.nio.file.FileSystemException: /usr/share/elasticsearch/data/elasticsearch/nodes/0/indices/rule_group_result_201804/1/index: Input/output error
```

错误原因：使用非 root用户启动ES，而该用户的文件权限不足而被拒绝执行。

解决方法： chown -R 用户名:用户名  文件（目录）名

例如： chown -R abc:abc searchengine
再启动ES就正常了

## 2、容器创建时端口已经分配

```
ERROR: for elasticsearch  Cannot start service elasticsearch: driver failed programming external connectivity on endpoint elasticsearch (f71ea6fc7387ac18b67963db17881427f8ced3b5a0cf2dca65f0290b1a8e1066): Bind for 0.0.0.0:9300 failed: port is already allocated
```

解决办法: 1、docker ps -a 找到占用端口的容器 docker stop  container

​			发现关不掉container   用docker rm f  container-name 强行杀掉容器

​		  2、netstat -lntp |grep  端口         过滤被占用端口所分配的进程

​			kill   进程号（PID）

## 3、集群健康值未连接

​	1、 vim  elasticsearch.yml

​		http.cors.enabled: true

​		http.cors.allow-origin: "*"

​	2、elasticsearch-head下Gruntfile.js

​	connect: {
        server: {
            options: {
                hostname: '0.0.0.0',
                port: 9100,
                base: '.',
                keepalive: true
            }
        }

​	}

##     3、查看未分配节点状态
```
curl -XGET 172.16.0.32:9200/_cat/shards?h=index,shard,prirep,state,unassigned.reason| grep UNASSIGNED
```

## 4、修改节点索引副本数量

2.x版本

```
curl -XPUT 172.16.0.32:9200/accounting_office/_settings -d'{"number_of_replicas": 1}'
```

curl -XPUT '172.16.0.32:9200/_cluster/settings'-d '{ "transient":   {"cluster.routing.allocation.enable" : "all"    } }' 

```
curl -XPUT http://172.16.0.32:9200/_settings?pretty -d '
{
  "index" : {
    "number_of_replicas' : 0
  }
}'
```

6.5.4版本

```
curl -X PUT "localhost:9208/123/_settings" -H 'Content-Type: application/json' -d'
{
    "index" : {
        "number_of_replicas" : 0
    }
}'
```

## 5、分片未分配的原因主要有：

1）INDEX_CREATED：由于创建索引的API导致未分配。
2）CLUSTER_RECOVERED ：由于完全集群恢复导致未分配。
3）INDEX_REOPENED ：由于打开open或关闭close一个索引导致未分配。
4）DANGLING_INDEX_IMPORTED ：由于导入dangling索引的结果导致未分配。
5）NEW_INDEX_RESTORED ：由于恢复到新索引导致未分配。
6）EXISTING_INDEX_RESTORED ：由于恢复到已关闭的索引导致未分配。
7）REPLICA_ADDED：由于显式添加副本分片导致未分配。
8）ALLOCATION_FAILED ：由于分片分配失败导致未分配。
9）NODE_LEFT ：由于承载该分片的节点离开集群导致未分配。
10）REINITIALIZED ：由于当分片从开始移动到初始化时导致未分配（例如，使用影子shadow副本分片）。
11）REROUTE_CANCELLED ：作为显式取消重新路由命令的结果取消分配。
12）REALLOCATED_REPLICA ：确定更好的副本位置被标定使用，导致现有的副本分配被取消，出现未分配

## 6、failed to read local state, exiting

## 7、Format version is not supported (resource SimpleFSIndexInput(path="/usr/share/elasticsearch/data/nodes/0/indices/zhiweinews_201708/_state/state-32.st")): 0 (needs to be between 1 and 1). This version of Lucene only supports indexes created with release 6.0 and later.

​      不支持格式版本（resourcesimplefsindexinput（path=“/usr/share/elasticsearch/data/nodes/0/indexs/zhiweinews_201708/_state/state-32.st”）：0（需要介于1和1之间）。此版本的Lucene仅支持使用6.0及更高版本创建的索引。

## 8、在head插件页面创建索引失败![1550630220592](C:\Users\ADMINI~1\AppData\Local\Temp\1550630220592.png)没反应

```
curl -XPUT ipaddr:9200/my_index
```

## 9、 max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]

```
max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
```

原因：无法创建本地文件问题,用户最大可创建文件数太小

解决方案：切到root 用户下

```
$ vim /etc/security/limits.conf 在文件的末尾添加下面的参数值：
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
```

前面的*符号必须带上，然后重新启动就可以了。执行完成后可以使用命令 ulimit -n 查看进程数

## 10、 max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

```
ERROR: [1] bootstrap checks failed
[1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
... ...
[2018-10-30T11:41:51,807][INFO ][o.e.x.m.j.p.NativeController] Native controller process has stopped - no new native processes can be started
```

原因：最大虚拟内存太小,需要修改系统变量的最大值。

解决方案：切换到root用户,修改配置sysctl.conf 增加配置值： vm.max_map_count=262144

```
$ vim /etc/sysctl.conf
vm.max_map_count=262144
```

执行命令

```
sysctl -p
```

## 11、max number of threads [1024] for user [es] likely too low, increase to at least [2048]

原因：无法创建本地线程问题,用户最大可创建线程数太小
解决方案：切换到root用户，进入limits.d目录下，修改90-nproc.conf 配置文件。

```
vi /etc/security/limits.d/90-nproc.conf
找到如下内容：
* soft nproc 1024
#修改为
* soft nproc 2048
```

## 12、system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk

原因：Centos6不支持SecComp，而ES6.4.2默认bootstrap.system_call_filter为true进行检测，所以导致检测失败，失败后直接导致ES不能启动。
详见 ：<https://github.com/elastic/elasticsearch/issues/22899>

解决方法：在elasticsearch.yml中新增配置bootstrap.system_call_filter，设为false，注意要在Memory下面:

```
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
```

然后重新启动ES服务 就可以了。

## 13、[ElasticSearch启动报错，bootstrap checks failed]

修改elasticsearch.yml配置文件，允许外网访问。

vim config/elasticsearch.yml
\# 增加

network.host: 0.0.0.0

启动失败，检查没有通过，报错

[2018-05-18T17:44:59,658][INFO ][o.e.b.BootstrapChecks    ][gFOuNlS] bound or publishing to a non-loopback address, enforcing bootstrap checks
ERROR: [2] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]

[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

 

 

[1]: max file descriptors [65535] for elasticsearch process is too low, increase to at least [65536]

编辑 /etc/security/limits.conf，追加以下内容；
\* soft nofile 65536
\* hard nofile 65536
此文件修改后需要重新登录用户，才会生效

 

[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

编辑 /etc/sysctl.conf，追加以下内容：
vm.max_map_count=655360
保存后，执行：

sysctl -p

重新启动，成功。

# 安装JAVA-JDK

第一步：下载安装包  官网（https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html）

linux64位 所以下载jdk-8u201-linux-x64.tar.gz

可以wget 链接地址   可以下载到本地在上传

解压：tar -xzfv  jdk-8u201-linux-x64.tar.gz    

之后会解压出一个名为jdk1.8.0_201的包 

第二部：安装  创建一个文件夹，作为你要安装的目录

$ mkdir /usr/java

$ mv jdk1.8.0_201    /usr/java

第三步：修改环境变量

$ vim /etc/profile

在文件末尾添加如下内容

```
export JAVA_HOME=/usr/java/jdk1.8.0_201
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib:$CLASSPATH
export JAVA_PATH=${JAVA_HOME}/bin:${JRE_HOME}/bin
export PATH=$PATH:${JAVA_PATH}:(后面的是之前其他程序的环境变量，以冒号隔开)
```

第四步：测试是否安装成功

1、使用javac命令，不会出现command not found错误

2、使用java -version，出现版本为java version "1.8.0_201"

![1550211526977](C:\Users\Administrator\Desktop\elk+.md)

3、echo $PATH，看看自己刚刚设置的的环境变量配置是否都正确

# 安装maven

安装Maven之前，首先要正确安装JDK，JDK确认无误后，首先进入Apache maven官网：https://maven.apache.org/，然后点击Download进入下载界面，或者直接进入下载界面：https://maven.apache.org/download.cgi，这里下载最新版本的maven 二进制包。

![1550212355973](C:\Users\ADMINI~1\AppData\Local\Temp\1550212355973.png)

下载完成之后上传至服务器，我们这里自定义安装位置为/usr/local/maven，安装命令操作如下：

```
$ tar -xvzf apache-maven-3.6.0-bin.tar.gz
$ mkdir /usr/local/maven
$ mv apache-maven-3.6.0 /usr/local/maven
```

　　然后配置环境变量，执行 vim /etc/profile 打开环境变量配置文件

　　在PATH后追加： :$MAVEN_HOME/bin

然后在最后添加两行代码，设置maven安装目录

```
MAVEN_HOME=/usr/local/maven/apache-maven-3.6.0
export MAVEN_HOME
```

然后保存退出，执行命令： source /etc/profile 使新增配置生效

　　然后执行下面命令确认maven安装成功：

```
mvn -v
```

![1550212646323](C:\Users\ADMINI~1\AppData\Local\Temp\1550212646323.png)

# 安装中文分词器ik

![1550547322321](C:\Users\ADMINI~1\AppData\Local\Temp\1550547322321.png)

以上是ik和es对应的版本号

在https://github.com/medcl/elasticsearch-analysis-ik/tags查找对应的ik版本

```
cd /tmp
wget https://github.com/medcl/elasticsearch-analysis-ik/archive/v5.5.0.zip
unzip v5.5.0.zip
cd elasticsearch-analysis-ik/
mvn package
cd target/releases/
cp elasticsearch-analyze-ik-*.zip elasticsearch/plugins/ik/
unzip elasticsearch-analyze-ik*zip
rm *zip
```

重启es（如果是容器，需要把plugins映射到/usr/share/elasticsearch/plugins）

测试：

最细致划分ik_max_word测试

```
curl 'http://172.16.0.32:9200/analyze/_analyze?analyzer=ik_max_word&pretty=true' -d '{"text":"我们是大数据开发技术人员"}'
```

粗略划分ik_smart测试

```
curl 'http://172.16.0.32:9200/analyze/_analyze?analyzer=ik_smart&pretty=true' -d '{"text":"我们是大数据开发技术人员"}'
```

# 安装pinyin分词器

![1550547512571](C:\Users\ADMINI~1\AppData\Local\Temp\1550547512571.png)

以上是pinyin分词器对应的es版本

在https://github.com/medcl/elasticsearch-analysis-pinyin/tags查找对应的版本

可以离线下载![1550547747515](C:\Users\ADMINI~1\AppData\Local\Temp\1550547747515.png)

上传到服务器，或者复制链接地址通过wget内部下载

后面步骤和前面安装ik分词器一样

如果解压后的版本和es版本不一致，可以在mvn package之前vim elasticsearch-analysis-pinyin-6.5.4/pom.xml

![1550566182963](C:\Users\ADMINI~1\AppData\Local\Temp\1550566182963.png)

将此处换成你的es版本号后再mvn编译。

# 索引的增删改查

1、创建索引   action.auto_create_index: true     es自动创建索引    

```
curl -X PUT "192.168.170.132:9200/test" -H 'Content-Type: application/json' -d'

{

    "settings" : {

        "number_of_shards" : 5,

        "number_of_replicas" : 0

    }

}

'

```

2、删除索引

```
curl -X DELETE "localhost:9200/twitter"

```

3、修改索引

```
curl -X PUT "172.16.0.32:9208/filebeat-*/_settings" -H 'Content-Type: application/json' -d'
{
    "index" : {
        "number_of_replicas" : 0
    }
}
'

```

4、查看索引

```
curl -X GET "localhost:9200/twitter"
```

5、设置重置

```
curl -X PUT "localhost:9200/twitter/_settings" -H 'Content-Type: application/json' -d'
{
    "index" : {
        "refresh_interval" : null
    }
}
'

```

3、若出现.monitoring-es-data     的索引

解决办法 vim elasticsearch.yml

```
xpack.monitoring.collection.enabled: true
xpack.monitoring.elasticsearch.collection.enabled: false
```

4、若出现 .monitoring-kibana-data 的索引

vim kibana.yml

```

```

1.编写清理脚本

```
仅支持这种形式的索引清理
green  open   k8s-stdout-2018.11.01                     PHEIz5NXSw-ljRvOP1cj9Q   5   1    3748246            0      3.2gb          1.6gb
green  open   recom-nginxacclog-2018.10.27              jl5ZBPHsQBifN0-pXEp_yw   5   1   52743481            0     20.8gb         10.4gb
green  open   nginx-json-acclog-2018.11.09              r6jsGcHWRV6RP2mY6zFxDA   5   1      95681            0     50.4mb         25.1mb
green  open   watcher_alarms-2018.11.09                 r_GS_GQGRoCgyOgiCFCuQw   5   1         24            0    323.4kb        161.7kb
green  open   .monitoring-es-6-2018.11.12               o8H2S-iERnuIAWQT7wuBRA   1   1       4842            0      3.2mb          1.6mb
green  open   client-nginxacclog-2018.11.02             FxXdLPpiSnuBtrJOB2BRsw   5   1  179046160            0    220.8gb        110.3gb
green  open   k8s-stderr-2018.11.16                     Vggw7iYCQ6OHtW-sphgGrA   5   1      68546            0     20.4mb         10.2mb
green  open   k8s-stderr-2018.10.21                     ZCeZZFRWSVyyYDKMjzPnbw   5   1      15454            0      5.3mb          2.6mb
green  open   watcher_alarms-2018.10.30                 VqrbMbnuQ3ChPysgnwgm2w   5   1         44            0      371kb        185.5kb
```

脚本  xxx.sh

```
#！/bin/bash
######################################################
# $Name:        clean_es_index.sh
# $Version:     v1.0
# $Function:    clean es log index
# $Author:      sjt
# $Create Date: 2018-05-14
# $Description: shell
######################################################
#本文未加文件锁，需要的可以加
#脚本的日志文件路径
CLEAN_LOG="/root/clean_es_index.log"
#索引前缀
INDEX_PRFIX=$1
DELTIME=$2
if [ "$DELTIME"x = ""x ]; then
   DELTIME=30
fi
 
echo "ready to delete index $DELTIME ago!!!"
 
#elasticsearch 的主机ip及端口
SERVER_PORT=10.2.1.2:9200
#取出已有的索引信息
if [ "$INDEX_PRFIX"x = ""x ]; then
     echo  "curl -s \"${SERVER_PORT}/_cat/indices?v\""
     INDEXS=$(curl -s "${SERVER_PORT}/_cat/indices?v" |awk '{print $3}')
else
     echo " curl -s \"${SERVER_PORT}/_cat/indices?v\""
     INDEXS=$(curl -s "${SERVER_PORT}/_cat/indices?v" |grep "${INDEX_PRFIX}"|awk '{print $3}')
fi
#删除多少天以前的日志，假设输入10，意味着10天前的日志都将会被删除
# seconds since 1970-01-01 00:00:00 seconds
SECONDS=$(date -d  "$(date  +%F) -${DELTIME} days" +%s)
#判断日志文件是否存在，不存在需要创建。
if [ ! -f  "${CLEAN_LOG}" ]
then
touch "${CLEAN_LOG}"
fi
#删除指定日期索引
echo "----------------------------clean time is $(date +%Y-%m-%d_%H:%M:%S) ------------------------------" >>${CLEAN_LOG}
for del_index in ${INDEXS}
do
    indexDate=$( echo ${del_index} | awk -F '-' '{print $NF}' )
    # echo "${del_index}"
    format_date=$(echo ${indexDate}| sed 's/\.//g')
    #根据索引的名称的长度进行切割，不同长度的索引在这里需要进行对应的修改
    indexSecond=$( date -d ${format_date} +%s )
    #echo "$SECONDS - $indexSecond = $(( $SECONDS - $indexSecond ))"
    if [ "$SECONDS" -gt "$indexSecond" ]; then
        echo "it will delete ${del_index}......."
        echo "${del_index}" >> ${CLEAN_LOG}
        #取出删除索引的返回结果
        delResult=`curl -s  -XDELETE "${SERVER_PORT}/"${del_index}"?pretty" |sed -n '2p'`
        #写入日志
        echo "clean time is $(date)" >>${CLEAN_LOG}
        echo "delResult is ${delResult}" >>${CLEAN_LOG}
    fi
done
```

2.加入crontab任务 进入crontab编辑模式 

crontab -e  

###### 在crontab编辑模式中输入  

##### 10 1 * * * sh /app/elk/es/es-index-clear.sh > /dev/null 2>&1 





# elasticsearch 模板

```
curl -X PUT "localhost:9200/_template/template_1" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["fileb*"],
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 0
  },
  "mappings": {
    "_doc": {
      "_source": {
        "enabled": false
      },
      "properties": {
        "host_name": {
          "type": "keyword"
        },
        "created_at": {
          "type": "date",
          "format": "EEE MMM dd HH:mm:ss Z YYYY"
        }
      }
    }
  }
}
'

```


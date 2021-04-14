 ## 关于hadoop(hdfs/yarn)的镜像封装

> 来源：https://www.kancloud.cn/huangzhenyou/shuoming/545497

|定稿人 | 定稿日期 | 系统环境|
| :--------:   | :-----:   | :----: |
|黄镇城 | 2018.1.18 | centos7 + docker1.13 + docker-compose1.16|

平时我们搭建hadoop集群的时候都是部署在物理机上，然而我们通过大量实验发现使用容器搭建hadoop集群不仅启动速度更快，而且资源利用率更高，本节主要讲述如何使用dockerfile封装hadoop，然后使用docker-compose编排集群。

### DockerFile源代码

```
# hadoop镜像封装
FROM openjdk:8u131-jre-alpine
MAINTAINER &lt;EWay>
# 设置一些系统环境变量（字符集和时区）
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8
ENV TZ=Asia/Shanghai

# 安装一些必需的软件包，同步时区 【标记*1】
RUN apk --update add wget bash tzdata \
    && cp /usr/share/zoneinfo/$TZ /etc/localtime \
    && echo $TZ > /etc/timezone
    
# 设置HADOOP相关的环境变量，这里我们使用2.7.4版本
ENV HADOOP_VERSION 2.7.4
ENV HADOOP_PACKAGE hadoop-${HADOOP_VERSION}
ENV HADOOP_HOME=/usr/hadoop
ENV PATH=$PATH:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin

# Hadoop安装
RUN wget http://www.apache.org/dist/hadoop/common/${HADOOP_PACKAGE}/${HADOOP_PACKAGE}.tar.gz && \
    tar -xzvf $HADOOP_PACKAGE.tar.gz && \
    mv $HADOOP_PACKAGE /usr/hadoop && \
    # 删除安装包，减少镜像大小
    rm $HADOOP_PACKAGE.tar.gz && \
    # 创建一系列HDFS需要的文件目录
    mkdir -p /root/hadoop/tmp && \
    mkdir -p /root/hadoop/var && \
    mkdir -p /root/hadoop/dfs/name && \
    mkdir -p /root/hadoop/dfs/data && \
    mkdir $HADOOP_HOME/logs && \
    # 格式化NameNode元数据
    $HADOOP_HOME/bin/hdfs namenode -format

WORKDIR $HADOOP_HOME
```
---
# 扩展
* 标记*1： 由于默认是美国时区，集群的机器时间得一致。所以安装软件同步本地时区
* namenode的格式化选择再构造镜像时进行，
  缺点：这样做后` -Ddfs.namenode.name.dir=/root/hadoop/dfs/name`  只能指定，不能灵活更改。 后期打算写shell脚根据截取字符来达到灵活更改。

dockerfile 编写完后我们就可以基于此构建镜像了
* -f 指定dockerfile文件，默认指定dockerfile，带参数后dockerfile可以命名其他名字

---

```powershell
# 构建Dockerfile 构建镜像
$ docker build -f Dockerfile -t hadoop:v1 .
```

### DockerFile代码注意事项
#### 源镜像选择
通常我们封装镜像时，一般都采用Centos或Ubuntu的源镜像，但是这样做有一个很大的缺点就是封装后的镜像体积太大，所以为了减少镜像体积，我们使用了体积小巧的alpine作为源镜像，同时考虑到HDFS和YARN都需要JAVA环境，所以最终我们使用了自带JDK的openjdk:8u131-jre-alpine。

> 注：最终镜像大小只有500M左右

#### 时区同步
源镜像默认是CTS时区，所以需要添加tzdata包获取各时区的相关数据，以更改为中国上海的时区。

> 注: tzdata安装更改时区后不可卸载，否则时区会变回CTS。

#### NameNode格式化
HDFS NameNode必须初始格式化后才可工作，所以这里直接在镜像中格式化，该条命令仅对NameNode生效，所以该条命令并不影响其他服务的使用。

> 优化: 这里可以使用脚本获取格式化路径，并判断是否需要格式化，这样做会使镜像变得更加灵活。


### 单独启动各服务容器
**本环节尤其重要的是如何解耦hadoop中的各个服务，达到每个容器启动一个服务进程，这样做一方面是为了解耦合，另一方面是为了更好的监控集群（一旦某容器即服务挂掉立马就可以看出来），如果是同一个容器启动多个服务的话，这样很难做到比较好的实时监控。**

* * * * *

1. 启动NameNode服务

```powershell
docker run -d --name namenode \
-p 50070:50070 -h namenode \
-v /root/hadoop/dfs/name:/root/hadoop/dfs/name \
hadoop:v1 hdfs namenode \
-Dfs.defaultFS=hdfs://0.0.0.0:9000 \
-Ddfs.namenode.name.dir=/root/hadoop/dfs/name \
-Ddfs.replication=3 \
-Ddfs.namenode.datanode.registration.ip-hostname-check=false \
-Ddfs.permissions.enabled=false \
-Ddfs.namenode.safemode.threshold-pct=0
```
> 注：使用HDFS命令后面的-D参数传递配置信息，这样可以使得镜像更加灵活；
> - dfs.namenode.datanode.registration.ip-hostname-check参数需要设置为false（默认为true），这样即使namenode中没有该datanode的DNS也可以完成注册。
> - dfs.namenode.safemode.threshold-pct参数的作用是设置namenode启动时可以容忍的数据块丢失比例，默认是99.9%。如果低于这一限制，那么namenode将一直处于安全模式，当分布式文件系统处于安全模式的情况下，文件系统中的数据是只读的。

---

# 扩展：
* 如下图所示，这样启动namenode进程的方法的好处在于可以自由更改配置，如果是写入hdfs-site.xml等文件，每次更改就必须重构hadoop镜像，再迭代更新中很笨重。（datanode/ResourceManager/NodeManager容器也是用带参数的方法启动。）
  ![](https://box.kancloud.cn/ef555f15d847de53ad399532e961b04a_651x216.png)
* **该思路是从$HADOOP_HOME/sbin和$HADOOP_HOME/bin目录下的可执行命令查看其shell源码。追溯分析后的方法。**

---

2. 启动DataNode服务并连接上NameNode

```powershell
docker run -d --name datanode -p 50075:50075 \
-v /root/hadoop/dfs/data:/root/hadoop/dfs/data \
--link namenode \
hadoop:v1 hdfs datanode \
-fs hdfs://namenode:9000 \
-Ddfs.datanode.data.dir=/root/hadoop/dfs/data \
-Ddfs.permissions.enabled=false
```
> 注：--link &lt;目标容器名> 该参数的主要意思是本容器需要使用目标容器的服务，所以指定了容器的启动顺序，并且在该容器的hosts文件中添加目标容器名的DNS。
> 

----
# 扩展
![](https://box.kancloud.cn/fb39e99253eb2baabafda504801421ac_635x161.png)

----

3. 启动ResourceManager服务
```powershell
docker run -d --name resourcemanager \
-p 8088:8088 -h ResourceManager \
hadoop:v1 yarn resourcemanager \
-Dfs.defaultFS=hdfs://namenode:9000 \
-Dmapreduce.framework.name=yarn \
-Dyarn.resourcemanager.hostname=ResourceManager \
-Dyarn.resourcemanager.webapp.address=0.0.0.0:8088
```


4. 启动NodeManager服务
```powershell
docker run -d --name nodemanager \
--link resourcemanager \
hadoop:v1 yarn nodemanager \
-Dfs.defaultFS=hdfs://namenode:9000 \
-Dmapreduce.framework.name=yarn \
-Dyarn.resourcemanager.hostname=ResourceManager \
-Dyarn.nodemanager.aux-services=mapreduce_shuffle \
-Dyarn.nodemanager.resource.memory-mb=2048 \
-Dyarn.nodemanager.resource.cpu-vcores=1 \
-Dyarn.nodemanager.vmem-check-enabled=false
```

### 使用docker-compose编排集群
虽然上面我们已经可以将HDFS和YARN的各个服务逐一启动，但是这样启动集群是不是有点麻烦呢，使用docker-compose编排工具可以帮助我们解决这个问题，完成一键启动整个集群的功能。


----
# 扩展：
* 一键盘式启动： 类似脚本功能，后期二次开发可以编写成前端一个按钮触发此功能。

---

```
namenode:
  image: hadoop:v1
  command: [
      "hdfs",
      "namenode",
      "-Dfs.defaultFS=hdfs://0.0.0.0:9000",
      "-Ddfs.namenode.name.dir=/root/hadoop/dfs/name",
      "-Ddfs.replication=3",
      "-Ddfs.namenode.datanode.registration.ip-hostname-check=false",
      "-Ddfs.permissions.enabled=false",
      "-Ddfs.namenode.safemode.threshold-pct=0"
      ]
  hostname: namenode
  expose:
    - 50070
    - 50090
    - 8020
    - 9000
  ports:
    - 50070:50070
  volumes:
      - /root/hadoop/dfs/name:/root/hadoop/dfs/name



datanode1:
  image: hadoop:v1
  command: [
      "hdfs",
      "datanode",
      "-fs", "hdfs://namenode:9000",
      "-Ddfs.datanode.data.dir=/root/hadoop/dfs/data",
      "-Ddfs.permissions.enabled=false"
      ]
  links:
    - namenode
  expose:
    - 50010
    - 50020
    - 50075
  volumes:
      - /root/hadoop/dfs/data1:/root/hadoop/dfs/data

datanode2:
  image: hadoop:v1
  command: [
      "hdfs",
      "datanode",
      "-fs", "hdfs://namenode:9000",
      "-Ddfs.datanode.data.dir=/root/hadoop/dfs/data",
      "-Ddfs.permissions.enabled=false"
      ]
  links:
    - namenode
  expose:
    - 50010
    - 50020
    - 50075
  volumes:
      - /root/hadoop/dfs/data2:/root/hadoop/dfs/data


resourcemanager:
  image: hadoop:v1
  command: [
      "yarn",
      "resourcemanager",
      "-Dfs.defaultFS=hdfs://namenode:9000",
      "-Dyarn.nodemanager.aux-services=mapreduce_shuffle",
      "-Ddfs.namenode.datanode.registration.ip-hostname-check=false",
      "-Ddfs.permissions.enabled=false -Dmapreduce.framework.name=yarn",
      "-Dyarn.resourcemanager.webapp.address=0.0.0.0:8088"
        ]
  hostname: resourcemanager
  expose:
    - 8030
    - 8031
    - 8032
    - 8033
    - 8040
    - 8042
    - 8088
  ports:
    - 8088:8088

nodemanager:
  image: hadoop:v1
  command: [
      "yarn",
      "nodemanager",
      "-Dyarn.resourcemanager.hostname=resourcemanager",
      "-Dyarn.nodemanager.resource.memory-mb=1024",
      "-Dyarn.nodemanager.aux-services=mapreduce_shuffle",
      "-Ddfs.namenode.datanode.registration.ip-hostname-check=false",
      "-Dmapreduce.framework.name=yarn",
      "-Ddfs.permissions.enabled=false"
      ]
  links:
    - resourcemanager
  expose:
    - 8030
    - 8031
    - 8032
    - 8033
    - 8040
    - 8042
    - 8088
```
这样编写好docker-compose文件后，就可以一键启动集群了

```
  # 一键启动docker-compose.yml编排的所有服务
  $ docker-compose -f docker-compose.yml up -d
  
  # 一键横向扩展nodemanager节点【标记2】
  $ docker-compose scale nodemanager=5
```
----

# 扩展
* 标记2：当遇到当前集群资源紧张时，可以一件扩展节点。


NaNundefined注意：如果有些服务想用scale命令扩展数量的话，是不能够有挂载点的，因为多个服务不能够映射同一个宿主机上的挂载路径，所以上面的服务中nodemanager是可以使用scale扩展的，而datanode就不可以，因为datanode需要使用挂载点持久化数据，这个问题在docker中并不能得到很好的解决，不过可以使用Kubernetes解决此问题。
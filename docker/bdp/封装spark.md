# 关于Spark集群的容器封装与部署
> https://www.kancloud.cn/huangzhenyou/shuoming/545497

|定稿人 | 定稿日期 | 系统环境|
| :--------:   | :-----:   | :----: |
|黄镇城 | 2018.1.28 | centos7 + docker1.13 + docker-compose1.16|

平时我们搭建hadoop集群的时候都是部署在物理机上，然而我们通过大量实验发现使用容器搭建hadoop集群不仅启动速度更快，而且资源利用率更高，本节主要讲述如何使用dockerfile封装hadoop，然后使用docker-compose编排集群。
### 封装Spark镜像
```dockerfile
  FROM openjdk:8u131-jre-alpine
  # openjdk:8u131-jre-alpine作为基础镜像，体积小，自带jvm环境。
  MAINTAINER &lt;eway&gt;
  # 切换root用户避免权限问题，utf8字符集，bash解释器。安装一个同步为上海时区的时间
  USER root
  ENV LANG=C.UTF-8
  RUN apk add --no-cache  --update-cache bash
  ENV TZ=Asia/Shanghai
  RUN apk --update add wget bash tzdata \
      && cp /usr/share/zoneinfo/$TZ /etc/localtime \
      && echo $TZ &gt; /etc/timezone
  # 下载解压spark
  WORKDIR /usr/local
  RUN wget "http://www.apache.org/dist/spark/spark-2.0.2/spark-2.0.2-bin-hadoop2.7.tgz" \
     && tar -zxvf spark-*  \
     && mv spark-2.0.2-bin-hadoop2.7 spark \
     && rm -rf spark-2.0.2-bin-hadoop2.7.tgz

  # 配置环境变量、暴露端口
  ENV SPARK_HOME=/usr/local/spark
  ENV JAVA_HOME=/usr/lib/jvm/java-1.8-openjdk
  ENV PATH=${PATH}:${JAVA_HOME}/bin:${SPARK_HOME}/bin

  EXPOSE 6066 7077 8080 8081 4044
  WORKDIR $SPARK_HOME
  CMD ["/bin/bash"]
```
dockerfile 编写完后我们就可以基于此构建镜像了
```powershell
# 构建Dockerfile构建镜像
$ docker build -f Dockerfile -t spark:v1 .
```
### DockerFile代码注意事项
#### 源镜像选择
通常我们封装镜像时，一般都采用Centos或Ubuntu的源镜像，但是这样做有一个很大的缺点就是封装后的镜像体积太大，所以为了减少镜像体积，我们使用了体积小巧的alpine作为源镜像，同时考虑到HDFS和YARN都需要JAVA环境，所以最终我们使用了自带JDK的openjdk:8u131-jre-alpine。


#### 时区同步
源镜像默认是CTS时区，所以需要添加tzdata包获取各时区的相关数据，以更改为中国上海的时区。
&gt; 注: tzdata安装更改时区后不可卸载，否则时区会变回CTS。

### 在本地启动服务

```powershell
# 启动 master 容器
$ docker run -itd --name spark-master -p 6066:6066 -p 7077:7077 -p 8080:8080 spark:v1 spark-class org.apache.spark.deploy.master.Master 

# 启动 worker 容器
$ docker run -itd  -P --link spark-master:worker01 spark:v1 spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077

# 启动 historyserver 容器
$ docker run -itd --name spark-history -p 18080:18080 \ 
-e SPARK_HISTORY_OPTS="-Dspark.history.ui.port=18080 \
-Dspark.history.retainedApplications=10 \
-Dspark.history.fs.logDirectory=hdfs://namenode:9000/user/spark/history" \
spark:v1  spark-class org.apache.spark.deploy.history.HistoryServer

# 运行spark程序
$ spark-submit --conf spark.eventLog.enabled=true \
--conf spark.eventLog.dir=hdfs://namenode:9000/user/spark/history \
--master spark://namenode:7077 --class org.apache.spark.examples.SparkPi ./examples/jars/spark-examples_2.11-2.0.2.jar 
```
---

# 扩展

* master节点相当于hadoop的namenode节点
* worker节点相当于hadoop的datanode节点
* historyserver节点，是为了记录我们运行的spark程序。通过浏览器访问` http://ip-addr:18080`。可以通过web详细的查询运行后的spark程序结果或过程。
* 运行spark程序例子
  ```shell
  spark-submit  \  #提交spark程序
  --conf spark.eventLog.enabled=true \   # 允许spark程序日志的记录，最后在ip:18080能查询到
  --conf spark.eventLog.dir=hdfs://namenode:9000/user/spark/history \ 
  # 将spark程序日志放在hadoop的【hdfs】分布式文件系统。也可以挂在在本地。
  但分布式文件系统因为分布式的原因他更安全，和能跨主机。    \
  --master spark://namenode:7077  \  # spark-submit提交spark程序到--master指定的master节点
  --class org.apache.spark.examples.SparkPi  ./examples/jars/spark-examples_2.11-2.0.2.jar 
  ```
---

### 使用docker-compose部署服务
```yaml
version: "2"
  master:
    image: spark:v1
    command: bin/spark-class org.apache.spark.deploy.master.Master -h master
    hostname: master
    environment:
      MASTER: spark://master:7077
      SPARK_MASTER_OPTS: "-Dspark.eventLog.dir=hdfs://namenode:9000/user/spark/history"
      SPARK_PUBLIC_DNS: master
    ports:
      - 4040:4040
      - 6066:6066
      - 7077:7077
      - 8080:8080

  worker:
    image: spark:v1
    command: bin/spark-class org.apache.spark.deploy.worker.Worker spark://master:7077
    hostname: worker
    environment:
      SPARK_WORKER_CORES: 1
      SPARK_WORKER_MEMORY: 1g
      SPARK_WORKER_PORT: 8881
      SPARK_WORKER_WEBUI_PORT: 8081
      SPARK_PUBLIC_DNS: master
    links:
      - master
    ports:
      - 8081:8081
      
 historyServer:
    image: spark:v1
    command: spark-class org.apache.spark.deploy.history.HistoryServer
    hostname: historyServer
    environment:
      MASTER: spark://master:7077
      SPARK_PUBLIC_DNS: master
      SPARK_HISTORY_OPTS: "-Dspark.history.ui.port=18080 -Dspark.history.retainedApplications=3 -Dspark.history.fs.logDirectory=hdfs://namenode:9000/spark/history"
    links:
      - master
    expose:
      - 18080
    ports:
      - 18080:18080
```
> 注：--link &lt;目标容器名&gt; 该参数的主要意思是本容器需要使用目标容器的服务，所以指定了容器的启动顺序，并且在该容器的hosts文件中添加目标容器名的DNS。

----

# 扩展
  ![](https://box.kancloud.cn/1499f5d8e072f13acef0389c4546b885_610x278.png)

```sh
  # 通过docker-compose一键启动spark集群
  $ docker-compose -f docker-compose.yml up -d
  
  # 一键扩展worker服务到5个
  $ docker-compose scale worker=5
```


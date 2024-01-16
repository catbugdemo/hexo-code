---
title: 部署Rocket-MQ
date: 2021-08-02 10:23:31
tags:
---

## 前言

- 学习一个组件的前提是去使用它
- rocketMQ 中存在几种部署模式：单机部署，多Master部署，多主多从，Dleger模式
- 因为目前是学习，所以只推荐 – 单机部署
- 单机部署两种方式
  1.本地下载运行 (推荐第一次安装时使用该方法)
  2.docker容器部署 (推荐第二次即以后使用该方法)
<!--more-->
## 1.本地安装

### 1.准备

- 安装 jdk 8+
- 安装 Maven 3.2.X+
- 安装 git

### 2. 安装 Rocket MQ

#### 1.下载,解压

```
COPY# 下载
$ wget https://archive.apache.org/dist/rocketmq/4.7.0/rocketmq-all-4.7.0-bin-release.zip

# 解压
$ unzip rocketmq-all-4.7.1-source-release.zip
```

#### 2.使用 Maven 编译RocketMQ 源码

```
COPY$ cd rocketmq-all-4.7.1-source-release
$ mvn -Prelease-all -DskipTests clean install -U
```

#### 3.修改 文件容量

ps: 因为 broker启动时的默认配置是 8g 8g 4g，如果你的服务器内存能达到以上需求，不需要进行如下的文件修改
ps: rocketmq 在 4.7 版本后以下文件都放在了 distribution 中.

```
COPY# 修改 bin/runserver.sh 文件
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:PermSize=128m -XX:MaxPermSize=320m"
 
#修改 bin/runbroker.sh 文件
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m"
 
# 修改 tools.sh 文件
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:PermSize=128m -XX:MaxPermSize=128m" 
```

### 3.启动 rocketmq

#### 1.启动 nameserver

```
COPY# 启动并重定向日志文件打印，后台运行
mkdir logs
nohup bin/mqnamesrv 1>logs/ng.log 2>logs/ng-err.log &

# jps命令 查看是否启动成功
$jps

29516 NamesrvStartup  # 意味着namesrv 成功启动
5982 Jps
```

#### 2.启动broker

```
COPY# 修改 broker.conf节点
vim conf/broker.conf
#添加 
namesrvAddr = 公网IP地址:9876
brokerIP1 = 公网IP地址

# 启动重定向日志，确认端口 ，启动自动创建节点,设定配置为指定文件，后台运行
nohup sh bin/mqbroker 1>/root/utils/rocketmq-all-4.7.0-bin-release/logs/mq.log -n 127.0.0.1:9876 autoCreateTopicEnable=true -c /root/utils/rocketmq-all-4.7.0-bin-release/conf/broker.conf &

# jps 查看
$jps
8448 Jps
29515 BrokerStartup #成功启动
29516 NamesrvStartup
```

### 4.模拟

#### 1.模拟生产者

```
COPY$export NAMESRV_ADDR=127.0.0.1:9876
$sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer 
```

#### 2.模拟消费者

```
COPYsh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

### 5.关闭

```
COPYmqshutdown namesrv       #关闭nameserver
mqshutdown broker          #关闭broker
```

### 6.通过控制台连接 rocketmq

#### 1.下载

```
COPYgit clone https://github.com/apache/rocketmq-externals.git
```

#### 2.配置

```
COPYcd rocketmq-exxternals/rocketmq-console/
# 修改maven项目的资源文件
$vim src/main/resource/application.properties

server.port=9877
rocketmq.config.namesrvAddr=127.0.0.1:9876
```

#### 3.打包运行

```
COPY# 查看地址
$pwd
~ /rocketmq-exxternals/rocketmq-console/

$mvn clean package -Dmaven.test.skip=true
$java -jar target/rocketmq-console-ng-2.0.0.jar >logs/console.log &
```

### 可能出现的问题

#### 1.nohup 空格即退出

> 没有配置好 bin 中的 sh 的 Xms 等最大容量

#### 2.closeChannel: close the connection to remote address[] result: true

> 没有开放 10909，10911端口
> [参考地址](https://github.com/apache/rocketmq/issues/568)

## 2. docker 容器部署

> [项目地址](https://github.com/apache/rocketmq-docker)

### 1.下载

```
COPY#下拉项目
git clone https://github.com/apache/rocketmq-docker.git

#进入
$cd rocketmq-externals-master/rocketmq-docker/image-build
$sh build-image.sh RMQ-4.7.1
$cd ..
$sh stage.sh 4.7.1
```

### 2.准备配置文件

#### 1.准备 broker.conf 文件

```
COPYvim broker.conf

brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
namesrvAddr = 公网ip:9876
brokerIP1 = 公网ip
```

#### 2.准备 docker-compose.yml 文件

```
COPYvim docker-rocketmq-compose.yml
 
version: '3.5'
services:
  rmqnamesrv:
    image: apacherocketmq/rocketmq:4.7.1
    container_name: rmqnamesrv
    ports:
      - 9876:9876
    volumes:
      - /opt/logs:/opt/logs
      - /opt/store:/opt/store
    command: sh mqnamesrv
    networks:
        rmq:
          aliases:
            - rmqnamesrv
  rmqbroker:
    image: apacherocketmq/rocketmq:4.7.1
    container_name: rmqbroker
    ports:
      - 10909:10909
      - 10911:10911
    volumes:
      - /opt/logs:/opt/logs
      - /opt/store:/opt/store
      - /home/data/rocketmq/broker.conf:/opt/rocketmq-4.7.1/conf/broker.conf 
    environment:
        TZ: Asia/Shanghai
        NAMESRV_ADDR: "rmqnamesrv:9876"
        JAVA_OPTS: " -Duser.home=/opt"
        JAVA_OPT_EXT: "-server -Xms512m -Xmx512m -Xmn512m"
    command: sh mqbroker rmqnamesrv:9876 -c broker.conf  autoCreateTopicEnable=true
    depends_on:
      - rmqnamesrv
    networks:
      rmq:
        aliases:
          - rmqbroker
  rmqconsole:
    image: styletang/rocketmq-console-ng
    container_name: rmqconsole
    ports:
      - 9877:8080
    environment:
        JAVA_OPTS: "-Drocketmq.namesrv.addr=rmqnamesrv:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=true"
    depends_on:
      - rmqnamesrv
    networks:
      rmq:
        aliases:
          - rmqconsole
networks:
  rmq:
    name: rmq
    driver: bridge
```

### 3.安装

```
COPYdocker-compose -f docker-rocketmq-compose.yml up -d
```

### 4.查看

- 查看： 设置的IP:9877 是否运行成功

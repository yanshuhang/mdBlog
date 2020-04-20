# win10使用wsl安装rocketmq

## 下载

``` linux
// 下载
wget https://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.7.0/rocketmq-all-4.7.0-bin-release.zip

// 解压
unzip rocketmq-all-4.7.0-bin-release.zip

// 顺便改个名字
mv rocketmq-all-4.7.0-bin-release rocketmq
```

另外需要安装jdk环境，推荐使用jdk8，高版本会有一些报错问题。  
需要配置JAVA_HOME等环境变量

``` shell
// 在/etc/profile文件最后添加如下
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
JRE_HOME=${JAVA_HOME}/jre
PATH=${JAVA_HOME}/bin:$PATH
export JAVA_HOME
export JRE_HOME
export PATH
```

## 修改配置

* 修改namesrv、broker的内存：默认的使用内存太大了

``` text
// bin/runserver.sh
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"

// bin/runbroker.sh
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m"
```

* 在conf/broker.conf中添加broker的ip，这里namesrv和broker在一台机器上就是127.0.0.1

``` text
// 添加ip
brokerIP1 = 127.0.0.1
```

## 启动

下面就可以跟着官方教程就可以了

* 启动name server

``` text
nohup sh bin/mqnamesrv &
```

* 启动broker

一定要加上`-c conf/broker.conf`

``` text
nohup sh bin/mqbroker -n localhost:9876 -c conf/broker.conf &
```

* 启动producer

``` text
export NAMESRV_ADDR=localhost:9876
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
```

* 启动consumer

``` text
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

* 关闭 servers

``` text
// 关闭broker
sh bin/mqshutdown broker

// 关闭name server
sh bin/mqshutdown namesrv
```

# 安装 RocketMQ

## 下载

```shell
cd /home/rain/software/rocketmq
wget https://archive.apache.org/dist/rocketmq/4.7.1/rocketmq-all-4.7.1-bin-release.zip

unzip rocketmq-all-4.7.1-bin-release.zip
ls -l

mv rocketmq-all-4.7.1-bin-release /home/rain/install/rocketmq/rocketmq-all-4.7.1
```



## 配置

### 修改日志配置

```shell
mkdir -p /home/rain/install/rocketmq/rocketmq-all-4.7.1/logs
cd /home/rain/install/rocketmq/rocketmq-all-4.7.1/conf && sed -i 's#${user.home}#/home/rain/install/rocketmq/rocketmq-all-4.7.1#g' *.xml
```



### 修改 NameServer JVM 参数

```shell
cd /home/rain/install/rocketmq/rocketmq-all-4.7.1/bin
vi runserver.sh

# 定位到如下代码
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"

# 修改　"-Xms -Xmx -Xmn"　参数
JAVA_OPT="${JAVA_OPT} -server -Xms512M -Xmx512M -Xmn256M -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```



### 修改 Broker 的配置文件

```shell
cd /home/rain/install/rocketmq/rocketmq-all-4.7.1/conf
vi broker.conf

# 使用如下配置文件
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

# 增加
storePathRootDir=/home/rain/install/rocketmq/rocketmq-all-4.7.1/store
storePathCommitLog=/home/rain/install/rocketmq/rocketmq-all-4.7.1/store/commitlog
namesrvAddr=127.0.0.1:9876
brokerIP1=192.168.80.8
brokerIP2=192.168.80.8
autoCreateTopicEnable=false
```



### 修改 Broker JVM 参数

```shell
cd /home/rain/install/rocketmq/rocketmq-all-4.7.1/bin
vi runbroker.sh 

#修改如下配置(配置前)
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"

#配置后
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn512m"
```



## 启动

### 启动 NameServer

```shell
cd /home/rain/install/rocketmq/rocketmq-all-4.7.1/bin

# jps NamesrvStartup
# ~/logs/rocketmqlogs/namesrv.log
nohup ./mqnamesrv &
```



### 启动 Broker

```shell
cd /home/rain/install/rocketmq/rocketmq-all-4.7.1/bin

# jps BrokerStartup
# ~/logs/rocketmqlogs/broker.log
nohup ./mqbroker -c ../conf/broker.conf &
```



### 查看集群状态

```shell
cd /home/rain/install/rocketmq/rocketmq-all-4.7.1/bin

sh ./mqadmin clusterList -n 127.0.0.1:9876
```



## 停止

```shell
sh mqshutdown broker

```



# 安装 RocketMQ-Console

## 下载

```shell
# 下载源码
cd /home/rain/software/rocketmq
wget https://github.com/apache/rocketmq-externals/archive/rocketmq-console-1.0.0.tar.gz

tar -xf rocketmq-console-1.0.0.tar.gz
```



## 配置

```shell
cd /home/rain/software/rocketmq/rocketmq-externals-rocketmq-console-1.0.0/rocketmq-console/src/main/resources
vi application.properties

# 修改
rocketmq.config.namesrvAddr=127.0.0.1:9876
```



## 编译

```shell
cd /home/rain/software/rocketmq/rocketmq-externals-rocketmq-console-1.0.0/rocketmq-console

mvn clean  package -DskipTests

cd /home/rain/software/rocketmq/rocketmq-externals-rocketmq-console-1.0.0/rocketmq-console/target
cp rocketmq-console-ng-1.0.0.jar ~/install/rocketmq/rocketmq-console
```



## 启动

```shell
cd  ~/install/rocketmq/rocketmq-console

# http://192.168.80.8:8080
nohup java -jar rocketmq-console-ng-1.0.0.jar &
```


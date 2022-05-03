[toc]

# RocketMQ集群搭建，通过NAT端口映射实现内外网通信

环境：2台物理机

- 物理机1：Ubuntu20.04 LTS，		公网ip1
- 物理机2：Windows10-VMware，  公网ip2
  - VM1：Ubuntu20.04 LTS           ip：192.168.107.128 
  - VM2：Ubuntu20.04 LTS           ip：192.168.107.129 
  - VM3：Ubuntu20.04 LTS           ip：192.168.107.130
- jdk版本：1.8，之前用的11，会有broker启动失败的问题，排查半天没找到哪里报错，换成open-jdk8好了。但是物理机1使用的是jdk11没有任何问题。

使用双主双从同步双写的方式搭建集群。

因为搭建集群需要物理机2上的VM与物理机1能够互相通信，我这里选择的方式是配置端口映射由宿主主机NAT转发到虚拟机然后实现互相之间的通信。需要注意的是broker除了会监听`listenPort`之外，还会监听`listenPort+1`和`listenPort-2`这两个端口，建立NAT映射时需要把这两个映射也建立好。

- **remotingServer：**监听listenPort配置项指定的监听端口，默认10911

- **fastRemotingServer：**监听端口值**listenPort-2**，即默认为10909

- **HAService：**监听端口为值为**listenPort+1**，即10912，该端口用于Broker的主从同步

## NAT端口映射配置（VMware）

![image-20220407153726953](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/nat_port_mapped.png)

![image-20220407153840191](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/nat_port_mapped_settings.png)

## 服务器ip地址

`/etc/hosts`：

```bash
# nameserver         name								port
202.199.6.118   rocketmq-nameserver1 #  9876
219.216.64.216  rocketmq-nameserver2 #  9876-192.168.107.128

# broker
202.199.6.118   rocketmq-master1		 #	10911
219.216.64.216  rocketmq-slave1			 # 	11913-192.168.107.128
219.216.64.216  rocketmq-master2		 #  10911-192.168.107.129
219.216.64.216  rocketmq-slave2			 #  12915-192.168.107.130
```

## 环境变量

`vim /etc/profile`，添加：

```bash
#set rocketmq
ROCKETMQ_HOME=/usr/local/rocketmq/rocketmq-4.9.3
PATH=$PATH:$ROCKETMQ_HOME/bin
export ROCKETMQ_HOME PATH
```

最后：

```bash
source /etc/profile
```

## 修改存储路径

```bash
mkdir /usr/local/rocketmq/store
mkdir /usr/local/rocketmq/store/commitlog
mkdir /usr/local/rocketmq/store/consumequeue
mkdir /usr/local/rocketmq/store/index
```

## 配置文件

### master1

修改部署`master1`机器的`broker-a.propertie`文件

```bash
vim broker-a.properties
```

配置如下：

```bash
# IP              NAME								 PORT
# 202.199.6.118   rocketmq-master1		 #	10911
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
# broker ip
brokerIP1=202.199.6.118
brokerIP2=202.199.6.118
#brokerIP2=
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=SYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```

### master2

修改部署`master2`机器的`broker-a.propertie`文件

```bash
vim broker-b.properties
```

配置如下：

```bash
# IP              NAME								 PORT
# 219.216.64.216  rocketmq-master2		 #  10911-192.168.107.129
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-b
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
brokerIP1=219.216.64.216
brokerIP2=192.168.107.129
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=SYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```

### slave1

修改部署`slave1`机器的`broker-a-s.properties`文件

```bash
# IP              NAME								 PORT
#219.216.64.216  rocketmq-slave1			 # 	10913-192.168.107.128
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=1
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
# broker ip
brokerIP1=219.216.64.216
brokerIP2=192.168.107.128
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=11913
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```

### slave2

修改部署`slave2`机器的`broker-b-s.propertie`文件

```bash
# IP              NAME								 PORT
# 219.216.64.216  rocketmq-slave2			 #  10915-192.168.107.130
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-b
#0 表示 Master，>0 表示 Slave
brokerId=1
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
# broker ip
brokerIP1=219.216.64.216
brokerIP2=192.168.107.130
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=12915
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```

## 启动集群

### 启动NameServer集群

在`NameServer`主机上执行：

```bash
cd $ROCKETMQ_HOME/bin
nohup sh mqnamesrv &
```

### 启动Broker集群

在部署`Broker`的集群上执行：

```bash
cd $ROCKETMQ_HOME/bin
nohup sh mqbroker -c /usr/local/rocketmq/rocketmq-4.9.3/conf/2m-2s-sync/[properties_file] &
```

注意把[properties_file]换成对应的配置文件：

- `master1`: `broker-a.properties`
- `master2`:`broker-b.properties`
- `slave1`:`broker-a-s.properties`
- `slave2`:`broker-b-s.properties`

```bash
# broker
202.199.6.118   rocketmq-master1		 #	10911
219.216.64.216  rocketmq-slave1			 # 	10913-192.168.107.128
219.216.64.216  rocketmq-master2		 #  10911-192.168.107.129
219.216.64.216  rocketmq-slave2			 #  10915-192.168.107.130
```

## 可视化工具

项目地址：[rocketmq-dashboard](https://github.com/apache/rocketmq-dashboard)

克隆到本地后编译成jar包运行：

```bash
mvn clean package -Dmaven.test.skip=true
java -jar target/rocketmq-dashboard-1.0.1-SNAPSHOT.jar
```

![image-20220407154136952](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/rocketmq_start_dashboard.png)

浏览器访问，可以看到集群已经启动起来了：

![image-20220407152737258](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/rocketmq_dashboard.png)
---
title: HBase部署方式
date: 2017-03-31 13:48:08
tags: HBase,Standalone Mode, Distributed Mode
---
## Standalone Mode
单节点独立部署
Standalone模式下，所有的HBase守护进程都运行在一个JVM中，像HMaster、一个HRegionServer，和ZooKeeper。

### 1. 下载并解压HBase安装包

### 2. 在hbase-env.sh中配置JAVA_HOME

### 3. 编辑hbase-site.xml，配置hbase.rootdir和hbase.zookeeper.property.dataDir
```
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>file:///home/testuser/hbase</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/home/testuser/zookeeper</value>
  </property>
</configuration>
```

<!--more-->

### 4. 启动
`bin/start-hbase.sh`是一个启动Hbase的非常方便的脚本。

### 5. 检查Web UI
访问http://localhost:16010 来查看HBase的Web UI

## 伪分布式部署
伪分布式意味着HBase仍旧运行在一台机器上，
但是每一个HBase守护进程（HMaster、HRegionServer、ZooKeeper）都作为一个独立的进程运行，
而Standalone模式下，所有守护进程都运行在一个JVM中。

### 1. 配置hbase-site.xml,添加如下属性
```
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
```
### 2. 启动HBase
`bin/start-hbase.sh `

### 3. 如果配置的hbase.rootDir是在HDFS，检查HDFS上是否有对应的hbase数据目录

### 4. 做一些数据库操作

```
create 'test', 'cf'
list 'test'
put 'test', 'row1', 'cf:a', 'value1'
put 'test', 'row2', 'cf:b', 'value2'
put 'test', 'row3', 'cf:c', 'value3'
scan 'test'
disable 'test'
drop 'test'
```

### 5. 启动停止backup HMaster
`bin/local-master-backup.sh start 2 3 5 `
HMaster会占用三个端口（默认是16010， 16020， 16030）
backup HMaster的端口偏移量在脚本后面作为参数
最多一个机器上可以启动10个HMaster
stop backup HMaster
` cat /tmp/hbase-testuser-2-master.pid |xargs kill `

### 6. 启动停止额外的HRegionServer
同backup HMaster类似
HRegionServer会占用两个端口（默认是16020， 16030，不过这两个端口已经被同一台机器上的HMaster占用，从HBase1.0.0开始，HMaster也是HRegionServer，所以这里的HRegionServer端口变为16200， 16300）
一台机器上，最多可以启动100个HRegionServer
`bin/local-regionservers.sh start 2 3 5 `
上面启动的HRegionServer端口针对16200/16300的偏移量分别是2、3、5
停止HRegionServer
`bin/local-regionservers.sh stop 2 `

### 7. 停止HBase
`bin/stop-hbase.sh`

## 完全分布式（Fully Distributed）
|Node Name|Master|RegionServer|ZooKeeper|
|---|---|---|---|
|node-a|yes|no|yes|
|node-b|backup|yes|yes|
|node-c|no|yes|yes|


### 1. 配置无密码SSH登陆
1. 在Master上生成ssh key ` ssh-keygen -t rsa `
2. 将生成的id_rsa.pub的内容，放到~/.ssh/authorized_keys文件的末尾（如果没有，需要自行创建）
3. 将Master上的公钥id_rsa.pub通过scp工具传到其他节点，并将内容添加到authorized_keys文件末尾
4. 测试Master能否无密码登陆本机和其他节点
5. 在backup HMaster节点做同样的操作，注意不要覆盖authorized_keys文件的已有内容
### 2. 配置Master
1. Master上运行HMaster和ZooKeeper，但是不运行HRegionServer
将conf/regionservers里面的node-a的信息去掉，添加node-b、node-c的hostname
2. 新建一个文件conf/backup-masters,内容填写node-b的hostname
3. 配置ZooKeeper
以下配置，让HBase在集群的每个节点上启动和管理一个ZooKeeper实例
```
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/Users/ligz/data/zookeeper</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>localhost</value>
    <description>usually comma seperated hostname list</description>
  </property>
```

### 3. 准备node-b， node-c
将node-a上的hbase安装目录，通过scp工具传到node-b和node-c上

### 4. 启动集群，并测试
启动前确保各个节点上的HBase相关进程已经停止（HMaster、HRegionServer、HQuorum）
在node-a上执行 `bin/start-hbase.sh`, 启动集群，观察控制台输出。
控制台顺序输出：启动ZooKeeper，启动Master，之后是RegionServer，最后是Backup Master
登陆各个节点，通过jps查看是否有相应的进程

### 5. 通过浏览器查看Web UI
Master访问端口16010（HBase0.98.x之前的版本，端口是60010）
RegionServer访问端口16030（Hbase0.98.x之前的版本，端口是60030）

## 部署模式
正如上面介绍的，分为Standalone和Distributed两种

### Standalone模式
所有的HBase守护进程都运行在一个JVM中。默认情况下，Standalone模式使用本地文件系统进行持久化操作。
如果想把数据持久化到HDFS上，需要修改hbase-site.xml中的两个配置项
1. hbase.rootdir修改为HDFS路径hdfs://ip:port/hbase
2. hbase.cluster.distributed设置为false

### Distributed模式
分为伪分布式和完全分布式，命名源自Hadoop里的伪分布式和分布式部署。
伪分布式可以持久化到本地文件系统或者HDFS。完全分布式只支持HDFS。
伪分布式是完全分布式模式的所有进程运行在一个节点上，仅仅用来在HBase上测试和建立原型。
不要在生产环境使用这个模式，也不要用此模式对HBase进行性能评估

完全分布式部署（Fully-Distributed），适用于生产环境。
采用完全分布式部署，HBase的守护进程运行在集群的每个节点上。
完全分布式部署需要修改配置项为***hbase.cluster.distributed***为true，***hbase.rootdir为HDFS路径***。
***conf/regionservers***里面配置集群中的RegionServer列表，每行一个hostname。
ZooKeeper，可以使用HBase自己管理的ZooKeeper集群（默认），也可以使用独立的ZooKeeper集群。
配置项为hbase-env.sh中的***HBASE_MANAGES_ZK***,默认是true，使用独立ZooKeeper集群时，修改此配置项为false。
使用HBase管理的ZooKeeper集群时，ZooKeeper的配置参数可以在hbase-site.xml中直接配置，
配置参数前缀使用hbase.zookeeper.property。
例如ZooKeeper的clientPort，在hbase-site.xml中配置项为hbase.zookeeper.property.clientPort
最少也需要配置`hbase.zookeeper.quorum`，默认只是绑定到了localhost，在完全分布式环境下并不适合，因为client连接不到。
***hbase.zookeeper.property.dataDir***最好明确配置为/tmp以外的目录。默认/tmp目录会被系统清理。

ZooKeeper***节点数目***，最好配置为***奇数***。这样可以更好地容忍节点失败。4个节点，容许1个节点fail，5个节点容许2个节点fail。
配置ZooKeeper服务器1G内存，可以的话，使用专属磁盘。
对于负担比较重的HBase集群，在RegionServers（DataNodes和TaskTrackers）之外的机器上运行ZooKeeper服务。
ZooKeeper版本，最好使用最新的稳定版本。从HBase1.0.0开始，需要3.4.x版本的ZK。
ZooKeeper维护设置，确保设置了***data dir 的cleaner***，要不然可能隔几个月会出现一些“有意思”的问题
如果ZooKeeper运行在数十万个日志的目录，zk会开始删除会话，这会导致Leader election不能完成。
虽然Leader Election很少进行，但是在机器丢失或者发生hipcup时，还是会随机发生。

使用独立ZooKeeper集群，
hbase-env.sh中的***HBASE_MANAGES_ZK***设置为false
hbase-site.xml中，设置***hbase.zookeeper.quorum***和***hbase.zookeeper.property.clientPort***

不想在HBase start/stop时start/stop ZooKeeper的话，可以单独执行ZooKeeper的start/stop
bin/hbase-daemons.sh {start,stop} zookeeper
同样需要HBASE_MANAGES_ZK设置为false

Hadoop Client端配置
如果修改了Hadoop急群众HDFS客户端的一些配置，需要使用下面三种方法中的一种来让HBase使用修改后的配置
1. 在hbase-env.sh中，向HBASE_CLASSPATH中添加HADOOP_CONF_DIR。
2. 向hbase的conf目录添加hdfs-site.xml的一份拷贝，或者更好地，使用软连
3. 如果只是几项HDFS配置修改的话，直接在hbase-site.xml中添加这几项修改。
举例来说，dfs.replication这个参数，如果修改为5，HBase不作处理的话，依旧是3.


## 运行和确认安装
确保HDFS、ZooKeeper已经成功运行。可以通过  `hdfs dfs -put from  to `  来测试HDFS是否正常工作
正常情况下，HBase不需要使用MapReduce和Yarn守护进程，所以这些不需要启动。
启动HBase命令，需要在HBASE_HOME目录下运行
```
bin/start-hbase.sh
```
HBase应该已经启动，可以检查$HBASE_HOME/logs里面的日志。
HBase提供了一个UI来展示重要的属性。默认是在Master节点的16010端口。
HBase RegionServer默认监听16020端口，并且在16030端口提供了HTTP服务。
HBase0.98之前的版本，HBase Master UI监听在60010，HBase RegionServers UI监听在60030
一旦HBase启动成功，可以执行一些create 、list、put、get、scan、disable、drop等命令，来查看HBase是否正常工作。
停止HBase命令 `bin/stop-hbase.sh`
停止操作需要等待片刻才能完成。如果集群由许多机器组成，会等待更长时间。
如果正在执行分布式操作，一定要等待HBase完全关闭后再去停止Hadoop守护进程。



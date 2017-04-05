#zoo-keeper、Kafka集群部署
##1.安装环境
	# lsb_release -dr
	Description:    Ubuntu 16.04.1 LTS
	Release:        16.04
	# uname -m
	x86_64
##2. 安装目录说明
	** 服务安装目录
	/application  
	** 服务配置文件、数据文件、进程ID和日志文件存放目录
	/data/kafka/conf
	/data/kafka/data
	/data/kafka/pid
	/data/kafka/logs
	** 服务软件和脚本存放目录
	/server/tools
	/server/scripts
	** 创建以上目录命令
	sudo mkdir -p /application /data/{kafka,zookeeper}/{conf,data,pid,logs} /server/{tools,scripts}
##3. zookeeper部署安装
1. zookeeper下载
	<pre># cd /server/tools 
	# wget http://mirrors.cnnic.cn/apache/zookeeper/zookeeper-3.4.6/zookeeper-3.4.6.tar.gz
	# tar xf zookeeper-3.4.6.tar.gz
	** 安装Java环境支持
	# sudo add-apt-repository ppa:webupd8team/java
	# sudo apt-get update
	# sudo apt-get install oracle-java8-installer</pre>	
2. 修改zookeeper配置文件
	<pre># mv /server/tools/zookeeper-3.4.6 /application
	# ln -s /application/zookeeper-3.4.6 /application/zookeeper
	# mv /application/zookeeper/conf/zoo_sample.cfg /data/zookeeper/conf/zoo.cfg
	# grep '^[a-z]' /data/zookeeper/conf/zoo.cfg
	tickTime=2000
	initLimit=10
	syncLimit=5
	dataDir=/data/zookeeper/data
	dataLogDir=/data/zookeeper/logs
	clientPort=2181
	autopurge.snapRetainCount=3
	autopurge.purgeInterval=24
	server.1=192.168.1.10:2888:3888
	server.2=192.168.1.11:2888:3888
	server.3=192.168.1.12:2888:3888
	skipACL=yes
	forceSync=no
	preAllocSize=64M
	globalOutstandingLimit=10000
	** server.1 这个1是服务器的标识也可以是其他的数字， 表示这个是第几号服务器，用来标识服务器，这个标识要写到快照目录下面myid文件里
	** 10.3.2.20等均为集群里的IP地址，第一个端口是master和slave之间的通信端口，默认是2888，第二个端口是leader选举的端口，集群刚启动的时候选举或者leader挂掉之后进行新的选举的端口默认是3888。默认端口均可在这里修改</pre>
3. 在zookeeper的每台服务器上创建my.id文件
	<pre>echo "1" > /data/zookeeper/data/myid
	echo "2" > /data/zookeeper/data/myid
	echo "3" > /data/zookeeper/data/myid</pre>
4. 启动服务
	<pre>/application/zookeeper/bin/zkServer.sh start /data/zookeeper/conf/zoo.cfg
	/application/zookeeper/bin/zkServer.sh status /data/zookeeper/conf/zoo.cfg
	** 将启动命令路径加入到系统环境变量中	
	# echo "export PATH=/application/zookeeper/bin/:$PATH" >> /etc/profile
	. /etc/profile
	zkServer.sh status</pre>
5.定期清理快照和日志文件
	<pre>#!/bin/bash 
 
	#snapshot file dir 
	dataDir=/data/zookeeper/data/version-2
	#tran log dir 
	dataLogDir=/data/zookeeper/logs/version-2

	#Leave 66 files 
	count=66 
	count=$[$count+1] 
	ls -t $dataLogDir/log.* | tail -n +$count | xargs rm -f 
	ls -t $dataDir/snapshot.* | tail -n +$count | xargs rm -f 

	#以上这个脚本定义了删除对应两个目录中的文件，保留最新的66个文件，可以将他写到crontab中，设置为每天凌晨2点执行一次就可以了。

	#zk log dir   del the zookeeper log
	#logDir=
	#ls -t $logDir/zookeeper.log.* | tail -n +$count | xargs rm -f</pre>
6. jvm调优  
	`echo export JVMFLAGS="-Xms2048m -Xmx2048m $JVMFLAGS" > /application/zookeeper/conf/Java.env`
	//调整完需重启
##4. kafka部署
1. kafka下载
	<pre># cd /server/tools
	# wget http://mirrors.hust.edu.cn/apache/kafka/0.10.1.1/kafka_2.10-0.10.1.1.tgz
	# tar xf kafka_2.10-0.10.1.1.tgz</pre>
2. 修改配置文件	
	<pre>mv /server/tools/kafka_2.10-0.10.1.1 /application
	# ln -s /application/kafka_2.10-0.10.1.1 /application/kafka
	# mv /application/kafka/config/server.properties /data/kafka/conf/
	# grep '^[a-z]' /data/kafka/conf/server.properties
	broker.id=1
	port=9092
	host.name=192.168.1.10
	num.network.threads=3
	num.io.threads=8
	socket.send.buffer.bytes=102400
	socket.receive.buffer.bytes=102400
	socket.request.max.bytes=104857600
	log.dirs=/tmp/kafka-logs
	num.partitions=1
	num.recovery.threads.per.data.dir=1
	log.retention.hours=1
	log.segment.bytes=1073741824
	log.retention.check.interval.ms=300000
	message.max.byte=5242880
	default.replication.factor=2
	replica.fetch.max.bytes=5242880
	zookeeper.connect=localhost:2181
	zookeeper.connection.timeout.ms=6000
	auto.create.topics.enable=true
	# useradd -M -s /usr/sbin/nologin kafka
	chown -R kafka:kafka /application/kafka/ /data/kafka/</pre>
3. 启动服务
	<pre># /application/kafka/bin/kafka-server-start.sh -daemon /data/kafka/conf/server.properties</pre>
4. 优化待补
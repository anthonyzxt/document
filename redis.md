#redis主从及sentital集群部署
---
##redis主从部署
1. 下载安装包  
	`cd /server/tools`  
	`wget http://download.redis.io/releases/redis-3.2.8.tar.gz`
2. 解压安装
	<pre># cd /server/tools
	# tar xf redis-3.2.8.tar.gz
	# mv redis-3.2.8 /application
	# ln -s /application/redis-3.2.8 /application/redis
	# cd /application/redis
	# make && make install</pre>
3. 修改配置文件
	<pre># cp redis.conf sentinel.conf /data/redis/conf/
	# cd /data/redis/conf/
	# grep '^[a-z]' redis.conf    
	bind 192.168.1.10
	protected-mode yes
	port 6379
	tcp-backlog 511
	timeout 0
	tcp-keepalive 300
	daemonize yes
	supervised no
	pidfile /data/redis/redis.pid
	loglevel notice
	logfile "redis.log"
	databases 16
	save 900 1
	save 300 10
	save 60 10000
	stop-writes-on-bgsave-error yes
	rdbcompression yes
	rdbchecksum yes
	dbfilename dump.rdb
	dir /data/redis/data
	masterauth 123456
	slave-serve-stale-data yes
	slave-read-only yes
	repl-diskless-sync no
	repl-diskless-sync-delay 5
	repl-disable-tcp-nodelay no
	slave-priority 100
	requirepass 123456
	maxmemory 8589934592
	maxmemory-policy volatile-lru
	appendonly no
	appendfilename "appendonly.aof"
	appendfsync everysec
	no-appendfsync-on-rewrite no
	auto-aof-rewrite-percentage 100
	auto-aof-rewrite-min-size 64mb
	aof-load-truncated yes
	lua-time-limit 5000
	slowlog-log-slower-than 10000
	slowlog-max-len 128
	latency-monitor-threshold 0
	notify-keyspace-events ""
	hash-max-ziplist-entries 512
	hash-max-ziplist-value 64
	list-max-ziplist-size -2
	list-compress-depth 0
	set-max-intset-entries 512
	zset-max-ziplist-entries 128
	zset-max-ziplist-value 64
	hll-sparse-max-bytes 3000
	activerehashing yes
	client-output-buffer-limit normal 0 0 0
	client-output-buffer-limit slave 256mb 64mb 60
	client-output-buffer-limit pubsub 32mb 8mb 60
	hz 10
	aof-rewrite-incremental-fsync yes</pre>
4. 启动redis服务
	/application/redis/src/redis-server /data/redis/conf/redis.conf
	/application/redis/src/redis-server /data/sentinel/conf/sentinel.conf --sentinel
5. 配置文件权限最小化
	chmod 600 /data/redis/conf/redis.conf 
	chmod 600 /data/sentinel/conf/sentinel.conf
##sentinel配置
	<pre># vi /data/sentinel/conf/sentinel.conf 
	daemonize yes
	protected-mode yes
	bind 192.168.1.11
	port 26379
	dir "/data/sentinel"
	sentinel myid 8904f76d87ffb9d088bb68204fabb86922f74ddf
	sentinel monitor mymaster 192.168.1.10 6379 2
	sentinel down-after-milliseconds mymaster 5000
	sentinel failover-timeout mymaster 15000
	sentinel auth-pass mymaster 123456</pre>
##redis几个危险命令做重命名处理
	//写配置文件也可，命令行处理也可
	rename-command FLUSHALL ""
    rename-command FLUSHDB ""
    rename-command KEYS ""
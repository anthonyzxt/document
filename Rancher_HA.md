#Rancher HA

>提示（重要）：由于rancher在部署的时候会有大量的数据写入库，在部署的时候一定要先部署好其中一台，所有节点都加入rancher后再做双活，否则在加入节点的时候会很慢
	
> MySQL采用主主模式+keepalive实现rancher-server双活

###1. MySQL双主模式配置
1. mysql配置：主1
	<pre>sudo cat /etc/mysql/mysql.conf.d/mysqld.cnf
	[mysqld_safe]
	socket          = /var/run/mysqld/mysqld.sock
	nice            = 0
	
	[mysqld]
	user            = mysql
	pid-file        = /var/run/mysqld/mysqld.pid
	socket          = /var/run/mysqld/mysqld.sock
	port            = 3306
	basedir         = /usr
	datadir         = /var/lib/mysql
	tmpdir          = /tmp
	lc-messages-dir = /usr/share/mysql
	skip-external-locking
	skip-name-resolve
	bind-address            = 10.3.5.20
	#default-storage-engine = myisam
	character_set_server = utf8
	collation-server = utf8
	transaction_isolation = READ-COMMITTED
	key_buffer_size         = 16M
	max_allowed_packet      = 16M
	thread_stack            = 192K
	thread_cache_size       = 8
	myisam-recover-options  = BACKUP
	max_connections        = 200
	query_cache_limit       = 1M
	query_cache_size        = 16M
	log_error = /var/log/mysql/error.log
	server-id               = 520
	log_bin                 = /var/log/mysql/mysql-bin.log
	expire_logs_days        = 1
	max_binlog_size   = 100M
	binlog_do_db            = cattle
	gtid_mode = on
	log-slave-updates = 1
	enforce_gtid_consistency = 1
	binlog_ignore_db        = mysql
	binlog_ignore_db        = information_schema
	replicate-do-db         = cattle
	replicate-ignore-db     = mysql
	replicate-ignore-db     = information_schema
	auto-increment-increment = 1 
	auto-increment-offset = 1
	symbolic-links = 0
	slave-skip-errors=all</pre>
2. mysql配置：主2
	<pre>sudo cat /etc/mysql/mysql.conf.d/mysqld.cnf
	[mysqld_safe]
	socket          = /var/run/mysqld/mysqld.sock
	nice            = 0
	
	[mysqld]
	user            = mysql
	pid-file        = /var/run/mysqld/mysqld.pid
	socket          = /var/run/mysqld/mysqld.sock
	port            = 3306
	basedir         = /usr
	datadir         = /var/lib/mysql
	tmpdir          = /tmp
	lc-messages-dir = /usr/share/mysql
	skip-external-locking
	skip-name-resolve
	bind-address            = 10.3.5.21
	#default-storage-engine = myisam
	character_set_server = utf8
	collation-server = utf8
	transaction_isolation = READ-COMMITTED
	key_buffer_size         = 16M
	max_allowed_packet      = 16M
	thread_stack            = 192K
	thread_cache_size       = 8
	myisam-recover-options  = BACKUP
	max_connections        = 200
	query_cache_limit       = 1M
	query_cache_size        = 16M
	log_error = /var/log/mysql/error.log
	server-id               = 521
	log_bin                 = /var/log/mysql/mysql-bin.log
	expire_logs_days        = 1
	max_binlog_size   = 100M
	binlog_do_db            = cattle
	gtid_mode = on
	enforce_gtid_consistency = 1
	binlog_ignore_db        = mysql
	binlog_ignore_db        = information_schema
	replicate-do-db         = cattle
	replicate-ignore-db     = mysql
	replicate-ignore-db     = information_schema
	auto-increment-increment = 2 
	auto-increment-offset = 1
	symbolic-links = 0
	slave-skip-errors=all</pre>
###2. keepalived配置

>keepalived安装： `sudo apt install keepalived -y`

1. 配置1
	<pre>sudo grep '[a-z]' /etc/keepalived/keepalived.conf 
	! Configuration File for keepalived
	global_defs {
	   notification_email {
	   notification_email_from m@m.loc
	   smtp_server 192.168.200.1
	   smtp_connect_timeout 30
	   router_id node1
	vrrp_instance VI_1 {
	    state MASTER
	    interface eno1
	    garp_master_delay 10
	    smtp_alert
	    virtual_router_id 51
	    priority 150
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 1111
	    virtual_ipaddress {
	        10.3.5.30/16 dev eno1 label eno1:1</pre>
2. 配置2
	<pre>sudo grep '[a-z]' /etc/keepalived/keepalived.conf 
	! Configuration File for keepalived
	global_defs {
	   notification_email {
	   notification_email_from m@m.oc
	   smtp_server 192.168.200.1
	   smtp_connect_timeout 30
	   router_id node2
	vrrp_instance VI_1 {
	    state BACKUP
	    interface eno1
	    garp_master_delay 10
	    smtp_alert
	    virtual_router_id 51
	    priority 100
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 1111
	    virtual_ipaddress {
	        10.3.5.30/16 dev eno1 label eno1:1</pre>
###3. Mysql双主实现
	<pre># 在主1上创建对应的rancher-server库:
	CREATE DATABASE IF NOT EXISTS cattle COLLATE = 'utf8_general_ci' CHARACTER SET = 'utf8';
	GRANT ALL ON cattle.* TO 'cattle'@'%' IDENTIFIED BY 'cattle';
	GRANT ALL ON cattle.* TO 'cattle'@'localhost' IDENTIFIED BY 'cattle';
	# 在两个库上分别创建用于复制的用户：
	grant replication slave on *.* to rep@'IP' identified by '密码'；
	#先在主库2上创建1的主从复制，然后再创建主库1的主从复制，主2的操作：
	CHANGE MASTER TO
	MASTER_HOST='10.3.5.20',
	MASTER_PORT=3306,
	MASTER_USER='rep',
	MASTER_PASSWORD='repsondon',
	MASTER_AUTO_POSITION=1;
	#主1的changmaster:
	CHANGE MASTER TO
	MASTER_HOST='10.3.5.21',
	MASTER_PORT=3306,
	MASTER_USER='rep',
	MASTER_PASSWORD='repsondon',
	MASTER_AUTO_POSITION=1;</pre>
###4. 分别启动rancher-server,加入节点是时候使用keepalived的虚拟IP。
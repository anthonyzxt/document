#MySQL 5.7.17二进制包部署安装
1. 下载并解压
	<pre># wget http://mirrors.sohu.com/mysql/MySQL-5.7/mysql-5.7.17-linux-glibc2.5-i686.tar.gz
	# tar xf mysql-5.7.17-linux-glibc2.5-i686.tar.gz 
	# mv mysql-5.7.17-linux-glibc2.5-i686 /application/mysql-5.7.17
	# ln -s /application/mysql-5.7.17/ /application/mysql</pre>
#编译安装
1. 下载安装包
2. 安装需要用到的插件及创建相关目录  
	`apt install libncurses-dev libaio-dev cmake gcc g++ -y`
	`mkdir /data/mysql/{data,logs,tmp,undolog}` 
3. 编译参数
	<pre>cmake . -DCMAKE_INSTALL_PREFIX=/application/mysql-5.7.17 \
	-DMYSQL_DATADIR=/data/mysql/data \
	-DMYSQL_UNIX_ADDR=/data/mysql/tmp/mysql.sock \
	-DDEFAULT_CHARSET=utf8 \
	-DDEFAULT_COLLATION=utf8_general_ci \
	-DEXTRA_CHARSETS=gbk,gb2312,utf8,ascii \
	-DENABLED_LOCAL_INFILE=ON \
	-DWITH_INNOBASE_STORAGE_ENGINE=1 \
	-DWITH_FEDERATED_STORAGE_ENGINE=1 \
	-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
	-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
	-DWITHOUT_PARTITION_STORAGE_ENGINE=1 \
	-DWITH_FAST_MUTEXES=1 \
	-DWITH_ZLIB=bundled \
	-DENABLED_LOCAL_INFILE=1 \
	-DWITH_READLINE=1 \
	-DWITH_EMBEDDED_SERVER=1 \
	-DDOWNLOAD_BOOST=1 \
	-DWITH_BOOST=/server/tools/mysql-5.7.17/boost \
	-DWITH_DEBUG=0
	//安装
	make && make install
	</pre>
4. 初始化数据库
	`/application/mysql/bin/mysqld --initialize --user=mysql --basedir=/application/mysql --datadir=/data/mysql/data/`
	//**初始化完后有个随机密码，这个密码一会登陆会用到**
5. 修改配置文件
	<pre>cat /etc/my.cnf 
	[client]
	port = 3306
	socket = /data/mysql/mysql.sock
	
	[mysqld]
	########basic settings########
	server-id = 1 
	port = 3306
	user = mysql
	bind_address = 0.0.0.0
	autocommit = 0
	character_set_server=utf8mb4
	collation-server = utf8mb4_unicode_ci
	skip_name_resolve = 1
	max_connections = 800
	max_connect_errors = 3000
	socket = /data/mysql/mysql.sock
	pid-file=/data/mysql/mysql.pid
	basedir = /application/mysql
	datadir = /data/mysql/data
	transaction_isolation = READ-COMMITTED
	explicit_defaults_for_timestamp = 1
	join_buffer_size = 1M
	tmp_table_size = 2M
	tmpdir = /tmp
	max_allowed_packet = 8M
	sql_mode = "STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER"
	interactive_timeout = 1800
	wait_timeout = 1800
	read_buffer_size = 1M
	read_rnd_buffer_size = 16M
	sort_buffer_size = 2M
	########log settings########
	log_error = error.log
	slow_query_log = 1
	slow_query_log_file = slow.log
	log_queries_not_using_indexes = 1
	log_slow_admin_statements = 1
	log_slow_slave_statements = 1
	log_throttle_queries_not_using_indexes = 10
	expire_logs_days = 30
	long_query_time = 2
	min_examined_row_limit = 100
	########replication settings########
	#master_info_repository = TABLE
	#relay_log_info_repository = TABLE
	log_bin = bin.log
	sync_binlog = 1
	gtid_mode = on
	enforce_gtid_consistency = 1
	log_slave_updates
	binlog_format = row 
	relay_log = relay.log
	relay_log_recovery = 1
	binlog_gtid_simple_recovery = 1
	slave_skip_errors = ddl_exist_errors
	########innodb settings########
	innodb_page_size = 16384
	innodb_buffer_pool_size = 2G
	innodb_buffer_pool_instances = 8
	innodb_buffer_pool_load_at_startup = 1
	innodb_buffer_pool_dump_at_shutdown = 1
	innodb_lru_scan_depth = 2000
	innodb_lock_wait_timeout = 5
	innodb_io_capacity = 4000
	innodb_io_capacity_max = 8000
	innodb_flush_method = O_DIRECT
	innodb_file_format = Barracuda
	innodb_file_format_max = Barracuda
	innodb_log_group_home_dir = /data/mysql/logs
	innodb_undo_directory = /data/mysql/undolog/
	innodb_undo_logs = 128
	innodb_undo_tablespaces = 0
	innodb_flush_neighbors = 1
	innodb_log_file_size = 4M
	innodb_log_buffer_size = 16M
	innodb_purge_threads = 4
	innodb_large_prefix = 1
	innodb_thread_concurrency = 16
	innodb_print_all_deadlocks = 1
	innodb_strict_mode = 1
	innodb_sort_buffer_size = 6G 
	########semi sync replication settings########
	plugin_dir=/application/mysql/lib/plugin
	plugin_load = "rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"
	loose_rpl_semi_sync_master_enabled = 1
	loose_rpl_semi_sync_slave_enabled = 1
	loose_rpl_semi_sync_master_timeout = 5000
	
	[mysqld-5.7]
	innodb_buffer_pool_dump_pct = 40
	innodb_page_cleaners = 4
	innodb_undo_log_truncate = 1
	innodb_max_undo_log_size = 1G
	innodb_purge_rseg_truncate_frequency = 128
	binlog_gtid_simple_recovery=1
	log_timestamps=system
	#transaction_write_set_extraction=MURMUR32
	show_compatibility_56=on
	
	[mysqldump]
	quick
	max_allowed_packet = 2M
	
	[mysql]
	no-auto-rehash
	
	[mysqld_safe]
	log-error=error.log
	pid-file=/data/mysql/mysql.pid</pre>
6. mysql目录授权并设置开机启动
	<pre>chown -R mysql.mysql /application/mysql
	chown -R mysql.mysql /data/mysql/
	cp /application/mysql/support-files/mysql.server /etc/init.d/mysqld
	sed -i 's#datadir=#datadir=/data/mysyql/data#g' /etc/init.d/mysqld
	sed -i 's#basedir=#basedir=/application/mysyql#g' /etc/init.d/mysqld
	echo "export PATH=/application/mysql/bin:$PATH" >> /etc/profile
	</pre>
7. 登陆后修改密码  
	`sudo systemctl enable mysqld`  
	`/etc/init.d/mysqld start`
	`SET PASSWORD = PASSWORD('123456');`
##mysql主从复制
1. 在主库创建复制专用账户
	<pre>grant replication slave on *.* to rep@'192.168.1.%' identified by 'repsondon';
	flush privileges;</pre>
2. 备份服务器数据  
	`mysqldump -uroot -p -A -B -F -R -x --master-data=2 --events --set-gtid-purged=OFF|gzip > kuaicong.sql.gz`
3. 在从库导入备份好的数据
	gzip -d /tmp/kuai
4. 从库导入masterinfo信息
	CHANGE MASTER TO
	MASTER_HOST='192.168.1.10',
	MASTER_PORT=3306,
	MASTER_USER='rep',
	MASTER_PASSWORD='repsondon',
	MASTER_LOG_FILE='bin.000009',
	MASTER_LOG_POS=194;
5.	启动从库并观察
##	mysql半同步复制
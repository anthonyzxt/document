#mongodb副本集

##配置文件
	<pre># cat /data/mongodb/conf/mongodb.conf
	# mongod.conf
	
	# for documentation of all options, see:
	#   http://docs.mongodb.org/manual/reference/configuration-options/
	
	# Where and how to store data.
	storage:
	  dbPath: /data/mongodb/data
	  journal:
	    enabled: true
	#  engine:
	#  mmapv1:
	#  wiredTiger:
	
	# where to write logging data.
	systemLog:
	  destination: file
	  logAppend: true
	  path: /data/mongodb/logs/mongod.log
	
	# network interfaces
	net:
	  port: 27017
	  bindIp: 127.0.0.1,192.168.1.10
	
	
	#processManagement:
	
	#security:
	#security:
	#  authorization: enabled
	#  keyFile: "/data/mongodb/mongod.pass"
	#operationProfiling:
	#replication:
	replication:
	  oplogSizeMB: 100
	  replSetName: sondon
	#sharding:
	
	## Enterprise-Only Options:
	
	#auditLog:
	
	#snmp:</pre>
##启动脚本命令
mongod -f /data/mongodb/conf/mongodb.conf --fork
##创建副本集
config = { _id:"sondon", members:[
... {_id:0,host:"192.168.1.10:27017"},
... {_id:1,host:"192.168.1.11:27017"},
... {_id:2,host:"192.168.1.12:27017"}]
... }
rs.initiate(config);
rs.status();

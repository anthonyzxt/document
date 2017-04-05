GlusterFS
---
##1. 环境准备
1. 系统版本信息
	<pre># lsb_release -a
	No LSB modules are available.
	Distributor ID: Ubuntu
	Description:    Ubuntu 16.04.1 LTS
	Release:        16.04
	Codename:       xenial 
	# uname -i
	x86_64</pre>
2. 关闭防火墙  
	<pre>systemctl stop ufw
	systemctl disable ufw</pre>
##2. 安装
1. 服务端安装  
	<pre>sudo add-apt-repository ppa:gluster/glusterfs-3.8
	sudo apt-get update
	sudo apt-get install glusterfs-server</pre>
2. 客户端安装  
	<pre>sudo add-apt-repository ppa:gluster/glusterfs-3.8
	sudo apt-get update
	sudo apt-get install glusterfs-client</pre>
##3. 配置分布式复制卷
	//分别在各节点创建存储目录
	mkdir /data/brick1
	//检查服务器是否启动
	systemctl status glusterfs-server.service
	//加入节点node1，master不必加
	root@master:~# gluster peer probe node1
	peer probe: success. 
	//查看节点信息
	root@master:~# gluster peer status      
	Number of Peers: 1	
	Hostname: node1
	Uuid: 570356c0-1ee4-4ce0-a03a-f7401b1c4573
	//增加分布式复制卷，相当于raid1，volume中brick所包含的存储服务器数量必须是repilca的倍数（>=2倍），兼顾两者的功能
	root@master:~# gluster volume create gv2 replica 2 master:/data/brick1 node1:/data/brick1 force         
	volume create: gv2: success: please start the volume to access data
	//启动复制卷
	root@master:~# gluster volume start gv2
	volume start: gv2: success
	//查看复制卷状态和信息，各个集群节点均可查看
	gluster volume status
	gluster volume info
##4. 客户端挂载
	//创建挂载目录,并确保安装了gluster-client,确保版本一致
	mkdir -p /mnt/gv
	glusterfs -V
	//在客户端各节点挂载
    mount -t glusterfs 10.3.2.21:gv2 /mnt/gv
##5. 利用GlusterFS为kubernetes提供PV（）和PVC（）
##6. 使用heketi为kubernetes提供动态存储
1. 文档来源  
	[https://github.com/gluster/gluster-kubernetes/tree/master/docs/examples/dynamic_provisioning_external_gluster](https://github.com/gluster/gluster-kubernetes/tree/master/docs/examples/dynamic_provisioning_external_gluster) 
2. 环境要求
	1. GlusterFS Cluster 2 nodes, each with at least 2 X 200GB（测试环境，大小试环境而定）raw devices
		* u64-node1(ubuntu 16.04 X86_64 10.3.3.84)
		* u64-node2(ubuntu 16.04 X86_64 10.3.3.85) 
	2. Heketi server/client
		* u64-node3(ubuntu 16.04 X86_64 10.3.3.86)
	3. 在heketi server上和client做好ssh认证
		<pre># ssh-keygen
		# ssh-copy-id root@u64-node1
		# ssh-copy-id root@u64-node2</pre>
3. 安装和heketi server和heketi-cli
	1. 下载地址heketi:  
		[https://github.com/heketi/heketi/releases/download/1.0.2/heketi-1.0.2-release-1.0.0.linux.amd64.tar.gz](https://github.com/heketi/heketi/releases/download/1.0.2/heketi-1.0.2-release-1.0.0.linux.amd64.tar.gz)
	2. 安装  
		<pre>tar xf heketi-1.0.2-release-1.0.0.linux.amd64.tar.gz
		cd heketi
		cp heketi heketi-cli /usr/bin</pre> 
	3. 配置heketi服务的配置文件，这里我们以ssh连接,配置文件内容如下 	 	
		<pre># cat heketi.json 
		{
        "_port_comment": "Heketi Server Port Number",
        "port" : "8080",

        "_use_auth": "Enable JWT authorization. Please enable for deployment",
        "use_auth" : false,

        "_jwt" : "Private keys for access",
        "jwt" : {
                "_admin" : "Admin has access to all APIs",
                "admin" : {
                        "key" : "My Secret"
                },
                "_user" : "User only has access to /volumes endpoint",
                "user" : { 
                        "key" : "My Secret"
                }
        },

        "_glusterfs_comment": "GlusterFS Configuration",
        "glusterfs" : {

                "_executor_comment": "Execute plugin. Possible choices: mock, ssh",
                "executor" : "ssh",

                "_db_comment": "Database file name",
                "db" : "heketi.db"
        },

        "_sshexec_comment": "SSH username and private key file information",
                "sshexec": {
                "keyfile": "/etc/heketi/heketi_key",
                "user": "root",
                "port": "22",
                "fstab": "/etc/fstab"
        },

        "_db_comment": "Database file name",
                "db": "/opt/heketi/heketi.db",

        "_loglevel_comment": [
              "Set log level. Choices are:",
              "  none, critical, error, warning, info, debug",
              "Default is warning"
            ],
            "loglevel" : "debug"
		}</pre>
	4. 启动heketi服务并测试  
		<pre># heketi -config heketi.json
		# curl http://u64-node3:8080/hello
		HelloWorld from GlusterFS Application</pre>
4. 配置heketi管理gluster
	1. 配置glusterfs集群拓扑文件
		<pre># cat topo.json
	{
	  "clusters": [
	    {
	      "nodes": [
	        {
	          "node": {
	            "hostnames": {
	              "manage": [
	                "u64-node1"
	              ],
	              "storage": [
	                "10.3.3.84"
	              ]
	            },
	            "zone": 1
	          },
	          "devices": [
	            "/dev/sdb",
	            "/dev/sdc"
	          ]
	        },
	        {
	          "node": {
	            "hostnames": {
	              "manage": [
	                "u64-node2"
	              ],
	              "storage": [
	                "10.3.3.85"
	              ]
	            },
	            "zone": 1
	          },
	          "devices": [
	            "/dev/sdb",
	            "/dev/sdc"
	          ]
	        }
	      ]
	    }</pre> 
	2. 使用heketi-cli导入glusterfs集群配置文件
		
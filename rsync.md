#rsync
1. 配置文件
	<pre># vi /etc/rsyncd.conf
	uid = rsync
	gid = rsync
	max connections = 100
	use chroot = no
	timeout = 100
	pid = /var/run/rsyncd.pid
	lock file = /var/run/rsyncd.lock
	log file = /var/log/rsyncd.log
	[backup]								
	path = /backup/
	ignore errors
	read only = false						
	list = false							
	hosts allow = 192.168.1.0/24			
	auth users = rsync_backup
	secrets file = /etc/rsyncd.passwd
</pre>
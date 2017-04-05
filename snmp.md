#snmpd

1. 报错日志：  
	snmpd[4872]: error on subcontainer 'ia_addr' insert (-1)
	解决方法：
	sed -i 's/Lsd/LS6d/g' /etc/default/snmpd
	service snmpd restart 或systemctl restart snmpd

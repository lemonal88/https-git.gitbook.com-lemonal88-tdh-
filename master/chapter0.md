# 第一章 Transwarp Manager的安装



##安装前准备

A、将linux主机的IP设置成静态IP，如192.168.1.200

B、在/etc/hosts文件中添加主机名，添加在最后一行，如192.168.1.200 dhc-1(注意hostname不支持使用'_'、'.')

C、使用chkconfig iptables off关闭防火墙

D、在/mnt目录中创建disk1目录

E、设置系统时间为NTP网络时间(如date -s '2016-1-19 9:00:00')

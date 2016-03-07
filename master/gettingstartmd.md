# 附录一：POC实施前准备


##第一章 OS & Linux

1、检查操作系统及其版本

查看方法：
```
cat etc/issue
```

2、检查hostname,FQDN,DNS

查看方法：
```
hostname

hostname-f

cat /etc/resolv.conf
```

更改方法：


```
vim /etc/sysconfig/network

vim /etc/hosts

□ hostname只能是以字母和数字的组合(中间允许’-‘)，不能有“,” / ”.” / “_”等特殊字符

vim /etc/resolv.conf

```

3、检查机器硬件配置

查看方法：

```
磁盘：df -h
内存：free -g (free -m)
网络：ethtool eth0 \ ifconfig
CPU:cat /proc/cpuinfo
```

4、检查机器时间

查看方法：date

更改方法：date -s '2016-3-3 9:00:00'

5、需要了解的Linux命令：

- 文件／文件夹操作类：

    **cd、mkdir 、rm 、cp 、mv 、touch 、du -h --max-depth=1**

- 查看文本：
    **cat、less、tail、vim、vi**

- 查找类：
    **grep、find**

- 压缩解压缩：**tar、gzip、zip、unzip**

- 帮助类：**man**
- 时间类：**date**
- IO类：**iostat -mx 3**
- 权限相关类：**sudo -u、chown、chmod、chgrp**
- 端口连接类：**netstat -nlap、ping IP、telnet IP端口**
- 查看文件占用：**lsof**
- 启停服务：**etc/init.d/mysqld [start|stop|restart]**
- 网页类：**elinks http://192.168.1.210:60010**
- 挂载：**mount、unmount**

第二章 安装TDH的Checklist
=======
####环境检查：

1、操作系统版本CentOS6.3-6.5/REHL6.3-6.5/SureSP2-SP3/操作系统是否干净？

2、是否需要配置sudo用户安装TDH？

3、机器硬件配置CPU/MEM是否满足要求？/系统根分区大于300G？/千兆以上网络？

4、是否配置了SSD？

5、是否操作系统双盘RAID1,数据盘RAID0？

6、配置是否对称同构

- 磁盘同构：数据盘对应的每块磁盘是否一样大？（严禁大小磁盘混合做数据盘，例如300G/mnt/disk1,2.7T/mnt/disk2）
    
- 网络同构：每台机器网卡配置是否相同？
    
- CPU/内存大小是否同构

7、系统时间是否正确 （date -s '2016-1-20 17:00:00'）
    
8、确认网络解析是用/etc/hosts文件还是DNS server

- 推荐使用hosts文件解析
    
- 若用hosts文件解析，确保/etc/resolv.conf为空或隐掉。并保证/etc/nsswitch.conf中files解析在DNS解析之前
    
- 各节点尽可能在同一个网段当中
    
9、hostname只能是以字母和数字的组合（中间允许‘-’），不能有‘，’ ‘.’ ‘_’等特殊字符

####TDH安装前的检查：

1、是否配置了RACK？（实施一定要配置，机柜命名一定要以‘/’开头，如/default）

2、Zookeeper配置个数是否检查？（奇数个，10个节点以下3个，10-50个节点5个）

3、HDFS的一个目录配置是否只包含/mnt/disk*的数据盘，SSD是否排除在外？

4、Hmaster个数是否为奇数？（3个或5个）

5、YARN的2个目录配置是否只包含/mnt/disk*的数据盘，SSD是否排除在外？

6、YARN的vcore/Mem配置是否配置成了1个core对应2G内存？

7、Inceptor是否配置了HiveServer2（推荐Kerberos+LDAP HA模式）

8、 Inceptor的fastdisk是否配置了SSD？

9、Inceptor的localdir配置里是否只包含/mnt/disk*,SSD是否排除在外？

10、Inceptor的资源配置是否合理？每个core是否都分配了2.5-2G内存？

####HA和安全配置:是否配置了HA？

1、KDC HA？

2、LDAP HA？

3、Metastore Mysql HA？

4、Resource Manager HA？

5、Kerberos条件下4.2版本的HDFS有token过期的问题，安装好后是否改对了配置？


##第三章 各组件配置和日志路径

配置：

- /etc/[组件名] 1

日志：

- /var/log









    


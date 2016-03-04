# 第一章：Transwarp Manager的安装



##安装前准备

A、在/etc/hosts文件中添加主机名，添加在最后一行，如192.168.1.200 dhc-1(注意hostname不支持使用'_','.')，配置完成后可以互相ping下，如果ping不通，请检查/etc/hosts文件和静态IP的设置

B、使用chkconfig iptables off关闭防火墙

C、在/mnt目录中创建disk1目录（若配有SSD固态硬盘还需创建randisk目录)

D、设置系统时间为NTP网络时间(如date -s '2016-1-19 9:00:00')



##安装步骤：

1、进入/mnt/disk1目录

![](1.png)
 
 
 2、使用root用户解压其中的transwarp安装包并安装
 ```
 >tar -zxvf transwarp-4.2.2-19029-zh.el6.x86_64.tar.gz
 >cd transwarp
 >./install
 
 ```
![](2.png)
 
 3、安装完成后，会自动弹出界面，依次选择Accept→选择网卡→默认端口8180→删除已有yum资源库→create new repository→Use ISO File→选择/mnt/disk1中的CentOS6.5安装包
 
 ![](3.png)
 ![](4.png)
 ![](5.png)
 ![](6.png)
 ![](7.png)
 ![](8.png)
 ![](9.png)
 
 
 4、安装好Centos6.5以后，打开chrome浏览器，输入安装Manager的本地节点ip地址加端口号8180，如192.168.1.200：8180，进行如下步骤操作：
 
（1）、输入admin、admin进入界面,为了方便多人对Mananger的操作，可以新建多个隶属于admin组的账户，同样可以操作服务器集群，避免了登入登出时被别的用户挤出。

（2）、填写集群名称（随意取名）

（3）、添加机柜（使用/rack1，/rack2......）指定

（4）、添加节点（可以使用［］来批量添加，如172.16.2.［68-70］）

（5）、输入root账号和密码进行确认设定

（6）、分配机柜，将刚刚的第一个节点分配到/rack1中，其他两个节点分配到/rack2中

（7）、选择需要/etc/hosts来确认网络解析

（8）、为了负载均衡，将YARN分配到/rack1中，Inceptor－server分配到/rack2中

5、安装组件和服务，按照左侧栏提示分别需要安装Zookeeper、HDFS、YARN、
Hyperbase、Inceptor－SQL，其他可以暂时不用安装

（1）Zookeeper：将全部节点都添加上（一定要为奇数），其他默认

（2）HDFS：记住两个重要目录即可，分别为dfs.namenode.name.dir和dfs.datanode.data.dir，分别在/home/hadoop节点下的hdfs_image和data目录下。另外需要特别注意的是，在安装HDFS过程中可能会遇到format namenode失败的现象，查看界面上的操作日志，可以看到报以下这个错误：

 ![](110.jpg)


这个错误的解决办法是：

a、在所有的JournalNode上，删除/hadoop/journal中所有的内容，然后执行service hadoop-hdfs-journalnode-hdfs1 restart

b、在所有NameNode节点上，清空dfs.namenode.name.dir配置的相应目录的所有内容

c、在所有DataNode节点上，清空dfs.datanode.data.dir配置的相应目录的所有内容

完成上述步骤后，可点击界面上的‘重试’键


（2）YARN：基础参数中配置yarn.nodemanager.resource.cpu-vcores的CPU核数，配置yarn.nodemanager.resource.memory-mb的内存大小，推荐配置为YARN的核数全给，内存给一半

CPU若不知道分几个核数，可以在命令行中执行``````cat /proc/cpuinfo | grep processor | wc -l

（3）HyperBase：配置master.memory内存大小，（若内存大小为8G，那么这里就应该是8G-YARN的                                 yarn.nodemanager.resource.memory-mb内存大小），Mastermemory相当于NN，Region server类似于DN，一般Master memory不耗费内存，主要Region server比较耗费内存

（4）Inceptor-SQL(SQL on spark)：
高级参数里面可以设置安全护栏，即hive.server.enable，值为FALSE不开启，值为                        TRUE后面服务就需要安装kerberos认证了，这项看具体实际需求。另外在资源分配选项中，executor有                                   Fixed（同构机器，每台机器配置差不多）和Ratio（异构机器，每台机器配置相差很大）两种，一般选择Fixed，下面的内核和内存千万不能超过YARN所设置的内核数和内存大小的值，因为Inceptor-SQL是从YARN那里申请                    资源的！推荐配置为内核数：内存＝1:2（1个内核配置2GB），
Inceptor server节点和Inceptor metastore节点需要安装在同一节点上，不然跨节点对表的操作会延迟会很高，Inceptor metastore存储的是表的信息，
记住Inceptor metastore节点的IP地址（即Inceptor server地址）因为使用sqoop服务要在metastore节点上操作mysql数据库（操作之前还需添加mysql的驱动）

6、确认安装后，在CLI下输入hive2登陆Inceptor的命令：

```
beeline -u jdbc:hive2://<Inceptor ip>:10000/  
```

（弃用没加安全认证的hive1登陆命令为：transwarp -t -h [Inceptor ip]

7、操作数据库
```
>show databases;
>use database;
>show tables;
>create table country(id int, name string);
```


##安装常见错误汇总：
1、format namenode出错，造成format namenode失败的原因是因为原TDH没有删除干净，在hadoop/namenode-dir/current里面有个锁，删除后可以使用命令etc/init.d/hadoop-hdfs-namenode start来启动namenode节点

2、如果HBase安装不成功，region server报红，首先cd /usr/lib/zookeeper/bin目录执行zkcli.sh -service [任意一台zookeeper主机ip]，再使用rmr /hyperbase命令将里面的hyperbase目录删除，要注意的是里面的原数据会丢失

3、Inceptor报黄就去YARN中检查



##常用命令：
1、Manager的相关命令可以使用：service --status-all来查看

2、显示hdfs集群的命令是：hadoop dfsadmin -report

或者sudo -u hdfs hdfs dfsadmin -reports

3、使HDFS中数据平衡的命令是：sudo -u hdfs hdfs balancer


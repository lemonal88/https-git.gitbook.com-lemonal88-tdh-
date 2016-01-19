# 第一章 TDH安装步骤

一、Transwarp Manager的安装：

安装前准备

a、将linux主机的IP设置成静态IP，如192.168.1.200

b、在/etc/hosts文件中添加主机名，添加在最后一行，如192.168.1.200 dhc-1(注意hostname不支持使用'_'、'.')

c、使用chkconfig iptables off关闭防火墙

d、在/mnt目录中创建disk1目录

e、设置系统时间为NTP网络时间(如date -s '2016-1-19 9:00:00')

1、进入/mnt/disk1目录


   
 ![](1.png)
 
 
 2、解压其中的transwarp安装包并安装
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
 
1、输入admin、admin进入界面

2、填写集群名称（随意取名）

3、添加机柜（使用/rack1，/rack2......）指定

4、添加节点（可以使用［］来批量添加，如172.16.2.［68-70］）

5、输入root账号和密码进行确认设定

6、分配机柜，将刚刚的第一个节点分配到/rack1中，其他两个节点分配到/rack2中

7、选择需要/etc/hosts来确认网络解析

8、为了负载均衡，将YARN分配到/rack1中，Inceptor－server分配到/rack2中






 
 
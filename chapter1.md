# 第一章 TDH安装步骤

一、Transwarp Manager的安装：
    
   1、将transwarp-4.2.2-19029-zh.el6.x86_64.tar.gz和CentOS-6.5-x86_64-bin-DVD1.iso两个安装包拷贝到/mnt/disk1中,disk1目录需要事先创建好，另外将系统时间改为当前时间。
 ![](1.png)
 
 
 2、解压其中的transwarp安装包并安装
 ```
 >tar -zxvf transwarp-4.2.2-19029-zh.el6.x86_64.tar.gz
 >cd transwarp
 >./install
 
 ```
 ![](2.png)
 
 3、安装完成后，会自动弹出界面，依次选择Accept→create new repository→use ISO
 选择CentOS6.5
 ![](3.png)
 ![](4.png)
 
 4、安装好Centos6.5以后，打开chrome浏览器，输入安装Manager的本地节点ip地址加端口号8180，如172.16.2.68：8180，进行如下步骤操作：
 
1、输入admin、admin进入界面

2、填写集群名称（随意取名）

3、添加机柜（使用/rack1，/rack2......）指定

4、添加节点（可以使用［］来批量添加，如172.16.2.［68-70］）

5、输入root账号和密码进行确认设定

6、分配机柜，将刚刚的第一个节点分配到/rack1中，其他两个节点分配到/rack2中

7、选择需要/etc/hosts来确认网络解析

8、为了负载均衡，将YARN分配到/rack1中，Inceptor－server分配到/rack2中
 
 
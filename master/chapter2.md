# 第三章 Sqoop安装与操作
##sqoop操作步骤：
（1）在Inceptor metastore节点服务器上安装sqoop服务
yum install sqoop

（2）由于Inceptor-SQL中metastore中已经安装了mysql，就不需要安装mysql了

（3）将mysql-connector-java-5.1.38tar.gz驱动包先解压
```
tar -zxvf mysql-connector-java-5.1.38tar.gz
```
（4）cd进刚刚解压后的目录，将里面的mysql-connector-java-5.1.38-bin.jar包copy到/usr/lib/sqoop/lib本地目录下

（5）在Inceptor Server节点上输入mysql，执行Grant操作：

----add user to mysql（username＝tdh，password＝123456），授权可以访问所有数据库
```
GRANT ALL PRIVILEGES ON *.* TO 'tdh'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
```
（若仅授权db1数据库里所有表，则可以这样指定)：
```
GRANT ALL PRIVILEGES ON db1.* TO 'tdh'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;）
```

（6）浏览mysql数据库
```
sqoop list-databases \
--username tdh \
--password 123456 \
--connect jdbc:mysql://172.16.1.15:3306/
```

（7）浏览mysql数据库中的表，db1为mysql中的一个数据库
```
sqoop list-tables \
--username tdh \
--password 123456 \
--connect jdbc:mysql://172.16.1.15:3306/db1
```

（8）从mysql————>HDFS上（import，将mysql中的db1数据库里面的表导入到/user/datadir，这里的datadir目录一定不要事先创建，不然会报错，语句执行的时候会自动创建目录的！）
```
sqoop import \
--username tdh \
--password 123456 \
--connect jdbc:mysql://172.16.1.15:3306/db1 \
--table country \
--target-dir /user/datadir
```

（9）从HDFS————>mysql表上（export）
```
sqoop export \
--username tdh \
--password 123456 \
--connect jdbc:mysql://172.16.1.15:3306/db1 \
--table cc \
--export-dir /user/testdir \
--staging-table tmptable
```



##注意事项
在执行导入导出数据时，可能由于yuan资源不足或者其他进程的占用，而一直停留在job作业等待处理中，
此时可以通过浏览器进入YARN中Resource Manager节点中的8088端口查看被占用的Application ID号，里面描述为Application master为常驻进程，不用
杀掉，再在shell中输入命令
```yuan -application -kill <Application ID>```
来杀死卡掉的进程，再运行上面的import、export语句。原因很简单，Inceptor-sql的常驻进程ApplicationMaster跑的是spark任务，非常消耗内存使用量，约为7-8G，所以在没有用到Inceptor-SQL的操作场景的时候就应该关闭该服务。


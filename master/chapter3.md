# Flume的安装和配置

1、安装Flume
```
yum install flume
```
    
2、在HDFS中创建flume目录，以存放来自本地的log日志文件（此/user/flume就是flume.conf中path的路径）
```
hadoop fs -mkdir /user/flume
```
    
3、在本地创建一个log日志文件或者txt文件（如在/tmp下创建一个a.txt文件，随意保存点内容）
    
4、进入Flume的默认配置路径修改flume.conf
```
>cd /usr/lib/flume/conf
>vi flume.conf
```
    

        
##flume.conf，将以下代码粘贴进去，保存退出

# Name the components on this agent
agent1.sources = source1
agent1.sinks = sink1
agent1.channels = ch1

# Describe/configure the source,下面的spoolDir一定要写本地存放log或txt的文件夹名，flume上传会将目录下所# 有log或txt文件都上传到HDFS中！！！！！
agent1.sources.source1.type = spooldir
agent1.sources.source1.spoolDir =/tmp
agent1.sources.source1.ignorePattern = .*dat.*


# Describe the sink，注意下面的path为Active Name Node，如果报standby错误，则去Manager中查看最新的
# Active Name Node
agent1.sinks.sink1.type = hdfs
agent1.sinks.sink1.hdfs.path = hdfs://<Active Name Node IP>:8020/user/flume/
agent1.sinks.sink1.hdfs.hdfs.rollInterval = 60
agent1.sinks.sink1.hdfs.hdfs.rollSize = 1024


# Use a channel which buffers events in memory
agent1.channels.ch1.type = file

# Bind the source and sink to the channel
agent1.sources.source1.channels = ch1
agent1.sinks.sink1.channel = ch1


5、退回到usr/lib/flume，执行以下flume上传命令：
bin/flume-ng agent -n agent1 -c conf -f conf/flume.conf -Dflume.root.logger=INFO,consoles

6、查看HDFS目录中/user/flume是否已经有刚刚上传的a.txt文件
hadoop fs -ls /user/flume
hadoop fs -cat /user/flume/*

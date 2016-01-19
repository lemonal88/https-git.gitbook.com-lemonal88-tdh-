# 第二章 Inceptor-SQL使用

##简介

Inceptor是一种交互式分析引擎，本质是一种SQL翻译器。Inceptor中一共可以操作四种类型的表结构：1、普通txt文本表 2、ORC表（Hive ORC格式）3、ORC事务表（可进行增删改查操作）4、Partition分区表（分为单值分区和范围分区）

##一、普通表导入数据
A、从HDFS导入数据

（1）创建HDFS数据目录，在本地创建一个存放数据的文件夹
```
hadoop fs -mkdir /user/datadir
```

（2）首先将本地path存放的数据文件put到HDFS目录中
```
hadoop fs -put  <path>/data.txt /user/datadir
```

（3）将上传进HDFS的文件load到Inceptor事先建立好的s3表中
```
load data inpath ‘user/datadir/data.txt’ into table s3；
```

（注意本步操作可能会报load数据没有权限，HDFS上的数据和表的权限不一致
使用：（sudo -u hdfs hadoop fs -chown -R hive /user/datadir）hive为owner名字，可以通过ESCE table;命令查看表中owner type字段名称）

B、从其他表导入

（1）将t3的表结构复制给t4，注意不复制数据
```
create table t4 like t3;
```

（2）查看
```
select * from t4;
```

（3）将t3表中的数据插入到t4表中
```
insert into table t4 select * from t3;
```


二、创建分区表

A、创建单值分区

(1)创建单值分区表（每创建一个单值分区表就会产生一个小文件，这里只有一个name值）
```
create table single_tbl(name string) partitioned by(level string)；
```

(注意后面的partition分区键和文本是无关的！文本只导入name！分区键是通过load语句中的level具体标识来指定的)
(2)把本地包含单列数据的txt文件put到HDFS中的/user/datadir目录中
hadoop fs -put /tmp/a.txt /user/datadir 
(3)将HDFS中的a.txt文件load到single_tbl单值分区表，即将a这个文档都设置成A标签
load data inpath ‘user/datadir/a.txt’ single_tbl partition(level='A');

B、创建范围分区表（用于避免全表扫描，快速检索，导入数据的方法也很少，只能通过从另一个表插入到范围表中，其产生原因是为了规避单值分区每创建一个表就会产生一个小文件，而范围分区则是每个分区存储一个文件）
（1）创建范围分区表rangepart
create table rangepart(name string)partitioned by range(age int)(
            partition values less than(30),
            partition values less than(40),
            partition values less than(50),
            partition values less than(MAXVALUE)
);
（注意分区表为左闭右开区间）
（2）将本地文件put到HDFS的user/datadir的目录中
hadoop fs -put /tmp/b.txt user/datadir
（3）创建外表，来将HDFS中的文件进行导入进来(外表是用来指定导入数据格式的，且drop外表时，HDFS上的数据还存在)
create external table userinfo(name string,age int) row format delimited fields terminated by ',' location 'user/datadir';
（4）将外表的数据插入到建立好的rangepart表中
insert into table rangepart select * from userinfo；
（5）查看插入分区表里的数据分布
show partitions rangepart;

三、创建分桶表（必须创建外表，只支持从外表导入数据去1，在分桶表中经常做聚合和join操作，速度非常快。另外分桶规则主要分为1、int型，按照数值取模，分几个桶就模几2、string型，按照hash表来分桶）
1、创建分桶表bucket_tbl
create table bucket_tbl(id int, name string)clustered by (id) into 3 buckets;
2、创建外表bucket_info
create external table bucket_info(id int, name string)row format delimited fields terminated by ',' location '/user/datadir';
2、将从本地txt文件put到HDFS中的表（如普通表），再load进外表中
load data inpath '/user/tdh/data/bucket-data' into table bucket_info;
2、设置分桶开关
set hive.enforce.bucketing=true；
3、插入数据
insert into table bucket_tbl select *from bucket_info;
按照取模后的大小排列：
insert into table bucket_tab1 select * from bucket_info
--建立ORC表（必须要分桶，可以做group by，order by操作）
（1）create table country（id int，country string）stored as orc
（2）create external table ex_tbl(id int,country string)
        row format delimited fields terminated by ','
        stored as textfile
        location '/user/tdh/externaltbl';
（3）insert into country select * from ex_tbl

--建立ORC事务表（必须要分桶，既可以单值插入，又可以通过外表插入）
（1）create table country(id int, country string) clustered by (id)into 3 buckets stored as orc tblproperties("transactional" = "true");
（2）create external表
（3）insert into country select * from external表


--------
Hyperbase表
（1）建立HBase表（用HBase和数据库做一个映射）
（2）建立HBase外表
create external table hbase_test……stored by ……with……
stored by指定HBase存储格式，with后面时序化和反序列化的！对应映射关系
（3）sql on HBase(若在Inceptor上对表进行一个操作，会在HBase同步)
key------>id
info------>name
info------>sex

注意事项：
1、HDFS不能直接直接load到Inceptor中的ORC事务表中，（只能load到普通表和ORC表中）要想在ORC事务表里插入数据有两种方法：a.建立一张外表，再将HDFS load进外表上，在insert into select * from external table    b.由于ORC事务表支持增删改查，所以可以使用单值插入语句insert into table country values（101，japan）
2、查看分区表的命令是show partitions [table名] 
3、使用命令hdfs dfs -ls /user/country
4、默认数据库存放位置
hdfs：//nameservice/inceptorsql1/user/hive/warehouse/
在Inceptor创建数据库时一般使用它的default默认数据库，若自己建立数据库请不要指定location，还有自己建立的数据库可能会因为权限不够而造成一些操作失败报错。
eg.（1）
create database ccc location ‘/user/ccc’；
create table ccc1；
上述语句建立的数据库位置为user/ccc/hive/ccc1

（2）
create table ccc2（a int）；
表示创建的ccc2表到默认路径user/ccc/hive/ccc2

（3）
create table ccc3（a int）location ‘user/ccc3’；
上述语句建立表的位置在user/ccc3
5、外表的作用是load导数据使用的，起到的是媒介作用，而ORC表则是做具体的操作的，外表一般是和ORC表配合使用的
6、分区表中的单值插入数据必须指定level
7、分桶中的桶大小，即一个文件大小一般为200M，处理效率最优，拿总文件大小除以200M就大概预估出分几个桶了
8、从HDFS中向Mysql中导入数据规定必须先在Mysql中创建临时表，先从HDFS的location目录下导入到tmp表中，再从tmp表导入到Mysql真正的表中
9、Flume需要先使用yum install flume命令安装，Flume的默认存放位置为/user/lib/flume/conf/flume.conf，vi进去后进行相应的修改，有两个位置需要注意，第一个是spoolDir后跟log所在HDFS中的文件夹名！切记，不是跟具体的log文件或者txt文件！（例如：spoolDir=/tmp/flume/），第二个是path后面是Active NameNode的HDFS路径
（例如：path＝hdfs：//172.16.2.77:8020/user/datadir），在flume.conf配置中默认指定缓冲区积攒到1k就写入HDFS中
10、养成在Inceptor中使用命令desc formatted <table名>；来查看各个表的底层结构和属性！！！
11、
    #创建内表，内表加载hdfs上的数据，会将被加载文件中的内容剪切走。
    #外表没有这个问题，所以在实际的生产环境中，建议使用外表。
12、hadoop fs：命令使用面最广，可以操作任何文件系统。
        hdfs dfs：命令只能操作HDFS文件系统相关。

##附录（示例代码）
```
--登录Inceptor
beeline -u jdbc:hive2://172.16.2.75:10000/

--DDL
--创建数据库(location 中文件的权限需和数据库owner一致)
--没有指定location，则数据库放在默认位置
--额外属性
create database db4
comment 'this is a database'
location '/user/test/'
with dbproperties('owener' = 'transwarp','time'='2016');

--在默认位置创建数据库，使用desc 查看数据库 
create database db2;
 
--删除数据库(数据库需为空)
drop database dbname

--描述数据库
describe database dbname;

--修改数据库
alter database db1 set dbproperties('owener'='hehe');


----------------------------创建简单text表--------------------------
--创建普通text表
create table t1(col1 string, col2 int);
create table t3(col1 int, col2 string)row format delimited fields terminated by ',' location '/user/datadir';
create table t2 like t1;
create table t4 as select col1 from t3;

--导入数据
--从hdfs导入数据(注意owner)
load data inpath '/user/datadir/data' into table t3;

--从其他表中插入
insert into table t4 select * from t3;



--创建分区表
--单值分区
create table part_t6(name string) partitioned by (level string);
--分区表导入数据，一次导入一个分区。分区键是从sql语句中定义，并不是从文件中导入。
--在本句中，如果文件有两列数据，则第二列是无效的。
load data inpath '/user/datadir/part-data1' into table part_t6 partition(level='A')//level变成A


--创建范围分区表(不支持从文件导入分区表，支持insert into select from)
create table rangepart_t7 (name string)partitioned by range(age int)(
partition values less than(30),
partition values less than(40),
partition values less than(50),
partition values less than(MAXVALUE)
);
--普通数据表
create external table user_info(name string, age int)row format delimited fields terminated by ',' location '/user/datadir';
load data inpath '/user/tdh/data/range-part1' into table user_info;
--通过普通数据表向范围分区表插入数据
insert into table rangepart_t7 select * from user_info;
select * from range_part_t6;
--查看分区信息
show partitions rangepart_t7;
--查看底层文件



-------------------------

--分桶表创建(数据只能通过insert插入，load无效)
--创建分桶表
--在windows下创建的文本文件是一\r\n换行，在unix下以\n换行，
--导入数据应该是unix格式的。
---以数字为bucket id
drop table if exists bucket_tbl;
create table bucket_tbl(id int, name string)clustered by (id) into 3 buckets;
--创建普通数据表
drop table if exists bucket_info;
create external table bucket_info(id int, name string)row format delimited fields terminated by ',' location '/user/datadir';
--导入数据到数据表中
load data inpath '/user/tdh/data/bucket-data' into table bucket_info;
--插入数据到分桶表中
set hive.enforce.bucketing = true;
insert into table bucket_tbl select *from bucket_info; 

---以字符串为bucket id,
drop table if exists bucket_tbl;
create table bucket_tbl(id int, name string)clustered by (name) into 3 buckets;
--创建普通数据表
drop table if exists bucket_info;
create external table bucket_info(id int, name string)row format delimited fields terminated by ',' location '/user/datadir';
--导入数据到数据表中
load data inpath '/user/tdh/data/bucket-data' into table bucket_info;
--插入数据到分桶表中
set hive.enforce.bucketing = true;
insert into table bucket_tbl select *from bucket_info;


----创建外表
drop table if exists ;
create external table ex_tbl(id int,country string)
row format delimited fields terminated by ','
stored as textfile
location '/user/tdh/externaltbl';
----------------------------------------------------------------------------------------

----建立ORC表
drop table if exists country;
create table country (id int, country string)stored as orc;

----建立ORC事务表
drop table if exists country;
create table  country(id int, country string) clustered by (id)into 3 buckets stored as orc tblproperties("transactional" = "true");
insert into table country select * from ex_tbl;
insert into table country values(100,'japan');
insert into table country values(101,'isis');


--创建分区分桶ORC表，分区字段不能用date类型，
drop table if exists country;
create table country (id int, country string) partitioned by(level string) clustered by (id)into 3 buckets stored as orc tblproperties("transactional" = "true");
--从外表插入数据
insert into country partition (level='A') select * from ex_tbl where id<5;
--单条插入
insert into table country partition (level='C') values(101,'isis');


--------------inceptor hyperbase------------
----建立hbase表 
create 'test','info'
put 'test','102','info:name','zhang'
put 'test','102','info:sex','male'

put 'test11','101','info:name','wang'
put 'test11','101','info:sex','female'
get 'test','101'
---建立hbase外表

create external table hbase_test(id string, name string,sex string)
stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
with serdeproperties('hbase.columns.mapping'=':key,info:name,info:sex') tblproperties('hbase.table.name'='test');


----------sql on hbase
create table sql_hbase11(id string, name string,sex string) stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
with serdeproperties("hbase.columns.mapping"=":key,info:name,info:sex") tblproperties("hbase.table.name"="test11");

insert into sql_hbase11(id,name,sex) values("11","zhang","male");




-----------------向hyperbase外表中导入1GB数据，
----先简历1GB数据普通表
create external table largedata(id string, name string, sex string)row format delimited fields terminated by ',' stored as textfile location '/user/zxh/'


---bulkload
--创建数据外表
--创建样本表
create table sampletable (id string, name string, sex string) row format delimited fields terminated by ',' stored as textfile;
insert into table sampletable select sample(30,id,name,sex)from largedata; 
--配置config.txt
--执行run.sh
```
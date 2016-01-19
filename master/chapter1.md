# 第二章 

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
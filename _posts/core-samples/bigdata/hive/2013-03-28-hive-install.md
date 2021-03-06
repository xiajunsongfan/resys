---
layout: post
title: "HIVE安装部署"
category : hive
tags : [hive, bigData, hadoop]
---
author : xiajun

1.下载HIVE安装包 [http://mirror.mel.bkb.net.au/pub/apache//hive/stable/hive-0.8.1.tar.gz](http://mirror.mel.bkb.net.au/pub/apache//hive/stable/hive-0.8.1.tar.gz "hive")

2.解压至hive目录 并将部门的所属人和所属组设为 hadoop(就是hadoop所使用的用户)

3.进入bin目录 修改hive-config.sh文件

	export HADOOP_HOME=/home/hadoop/hadoop-1.1.1
	export HIVE_HOME=/home/hadoop/hive-0.10.0 
	export JAVA_HOME=$JAVA_HOME  根据自己的情况配置自己的这三个配置自己。
4.如果要结合MYSQL就将MYSQL的驱动放入lib目录。

5.配置连接mysql的文件(新建)conf/hive-site.xml

<?prettify lang=xml linenums=true?>
<pre class="prettyprint linenums" id="quine" style="border:4px solid #88c">
<xmp>
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<property>
<name>javax.jdo.option.ConnectionURL</name>
<value>jdbc:</value>
</property>
<property>
<name>javax.jdo.option.ConnectionDriverName</name>
<value>com.mysql.jdbc.Driver</value>
</property>
<property>
<name>javax.jdo.option.ConnectionUserName</name>
<value>root</value>
</property>
<property>
<name>javax.jdo.option.ConnectionPassword</name>
<value>123456</value>
</property>

<property>
<name>hive.metastore.warehouse.dir</name><!-- 数据存放目录-->
<value>/user/hive/data</value>
</property>
<property>
 <name>hive.exec.scratchdir</name><!-- 临时目录--> 
<value>/home/xiajun/hadoop-data/hadooptemp/hive_tmp</value>
</property>
</configuration></xmp>
</pre>
当hive使用mysql作为元数据库的时候mysql的字符集要设置成latin1   default。
为了保存那些utf8的中文，要将mysql中存储注释的那几个字段的字符集单独修改为utf8。

修改字段注释字符集

	alter table COLUMNS modify column COMMENT varchar(256) character  set utf8;

修改表注释字符集

	alter table TABLE_PARAMS modify column PARAM_VALUE varchar(4000)  character set utf8;
6.启动hive  ./hive/bin/hive

7.常用命令

	show tables;//查看所有表
	show databases;

	create table test(id int,name string) comment '这是表注释'  row format delimited fields terminated by '\t'; // \t 是一行中列的分隔符 tab

	CREATE TABLE gdm.pokes (foo INT, bar STRING COMMENT '中文注释');//表默认建在default库上，gdm.pokes 表示表建在gdm库上

	create [external] table test(id int,name string) row format delimited fields terminated by '\t' LOCATION 'hdfs://xxx 指向必须是目录 如果不存在会自己创建'; external 是创建外部表的关键字

	drop table test;

	describe test; //查看test表的结构 简写 desc table

	describe extended test; //查看test表结构详细属性 

	desc formatted test; //查看test表的详细属性 

	alter table test rename to test2; 将表名test修改为test2

	alter table test2 add columns(age int); 给test2表添加字段

	ALTER TABLE invites ADD COLUMNS (new_col2 INT COMMENT 'a comment');

	alter table test2 change age age2 string;将test2表中的age字段修改为age2 并将其字段类型改为string

	load data local inpath '../data.txt' overwrite into table test2;导入数据到test2表中

	nohup ./hive --service hiveserver &  //hive后台启动
8.查询数据结构的处理

8_1.将select的结果放到一个的的表格中（首先要用create table创建新的表格）
	
	insert overwrite table test select uid,name from test2;

8_2.将select的结果放到本地文件系统中

	INSERT OVERWRITE LOCAL DIRECTORY  '/tmp/reg_3'  SELECT a.* FROM events a;

8_3.将select的结果放到hdfs文件系统中

	INSERT OVERWRITE DIRECTORY  '/tmp/hdfs_out' SELECT a.* FROM invites a WHERE a.ds='《DATE>';

8_4.将查询结构放入新建表中。

	create table test2 as select count(id) as count from test group by name order by count desc;

9.使用自定义inputformat

	create table test(id int,name string) row format delimited fields terminated by ' ' STORED AS inputformat 'com.MyInputformat'  outputformat 'com.MyOutputformat';

例子：

	create table score(name string, score map《string,int>)
	ROW FORMAT DELIMITED
	FIELDS TERMINATED BY '\t'
	COLLECTION ITEMS TERMINATED BY ','
	MAP KEYS TERMINATED BY ':';

    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY '\001'
    COLLECTION ITEMS TERMINATED BY '\002'
    MAP KEYS TERMINATED BY '\003'
    STORED AS TEXTFILE;
    [ROW FORMAT DELIMITED]关键字，是用来设置创建的表在加载数据的时候，支持的列分隔符。不同列之间用一个'\001'分割,集合(例如array,map)的元素之间以'\002'隔开,map中key和value用'\003'分割。

要入库的数据

	zhang '数学':80,'语文':89,'英语':95
	
	li    '语文':60,'数学':80,'英语':99

10.内部表和外部表

内部表：在创建完表之后将数据导入表中，数据存放在表所指定的hdsf中，当表被删除时数据也会被删掉。

外部表：在创建表时就指定表所使用的数据地址(就像连接引用一样，同样数据也是在hdfs中只不过是自己存放的位置)，当表被删除时，数据任然存放在hdfs原来的位置，但表结构会被删除。

11.表分区
hive可使用日期(dt)和时间(hour)分区。

	select name,dt from table where dt='2013-03-10' limit 1;
	show partitions table;//查看表分区信息
12.为mahout提供索引

	create table test_index as select b.pin,rank(b.pin) rank from (select distinct a.pin from test a)b group by b.pin;

13.常见错误
	Error while reading from task log url

http://slave3:50060/tasklog?taskid=attempt_201212192008_0014_m_000000_3&start=-8193

修改为：

http://slave3:50060/tasklog?attemptid=attempt_201212192008_0014_m_000000_3&start=-8193
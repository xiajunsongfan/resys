---
layout: post
title:  "hive-hbase整合"
date:   2013-08-14 19:03:11
category: bigData
tags : [bigData, hive, hbase, hadoop]
---
##hive-hbase整合
**author : xiajun**
-
**前提:**

1.Hadoop和Hbase都已经成功安装了

2.拷贝hbase-0.94.6.jar和zookeeper-3.4.5.jar，protobuf-java-2.4.0a.jar 到hive/lib下。

    注意：如果hive/lib下已经存在这两个文件的其他版本（例如zookeeper-3.3.2.jar），建议删除后使用hbase下的相关版本。

3.修改hive/conf下hive-site.xml文件，在底部添加如下内容：

	<property>          
	  <name>hive.querylog.location</name>          
	  <value>/usr/local/hive/logs</value>          
	</property>          
	                
	<property>         
	  <name>hive.aux.jars.path</name>          
	  <value>file:///usr/local/hive/lib/hive-hbase-handler-0.8.0.jar,file:///usr/local/hive/lib/hbase-0.90.4.jar,file:///usr/local/hive/lib/zookeeper-3.3.2.jar</value>         
	</property>


4.拷贝hbase-0.94.6.jar  protobuf-java-2.4.0a.jar 到所有hadoop节点(包括master)的hadoop/lib下。

5.拷贝hbase/conf下的hbase-site.xml文件到所有hadoop节点(包括master)的hadoop/conf下。

6.1单节点启动

	./bin/hive -hiveconf hbase.master=master:490001

6.2集群启动：

	./bin/hive -hiveconf hbase.zookeeper.quorum=node1,node2,node3

7.如何hive-site.xml文件中没有配置hive.aux.jars.path，则可以按照如下方式启动。

	bin/hive --auxpath /usr/local/hive/lib/hive-hbase-handler-0.8.0.jar, /usr/local/hive/lib/hbase-0.90.5.jar, /usr/local/hive/lib/zookeeper-3.3.2.jar -hiveconf hbase.zookeeper.quorum=node1,node2,node3

8.创建表

	create table h_test2(key int,name string,sex string,age int) stored by
	'org.apache.hadoop.hive.hbase.HBaseStorageHandler' with serdeproperties 
	("hbase.columns.mapping"=":key,stu:name,stu:sex,stu:age") TBLPROPERTIES ("hbase.table.name" = "h_test2");

注：hbase.table.name 定义在hbase的table名称  hbase.columns.mapping 定义在hbase的列族

二.使用sql导入数据

1) 新建hive的数据表:

	CREATE TABLE pokes (foo INT, bar STRING);
2)批量插入数据:

	hive> LOAD DATA LOCAL INPATH './examples/files/kv1.txt' OVERWRITE INTO TABLE pokes;

3)使用sql导入hbase_table_1:

	hive> INSERT OVERWRITE TABLE hbase_table_1 SELECT * FROM pokes WHERE foo=86; 

3.查看数据

	hive> select * from  hbase_table_1;  

这时可以登录Hbase去查看数据了

	.bin/hbase shell
	hbase(main):001:0> describe 'h_test'  
	hbase(main):002:0> scan 'h_test'  
	hbase(main):003:0> put 'h_test','100','stu:name','xxx'

这时在Hive中可以看到刚才在Hbase中插入的数据了。

4.hive访问已经存在的hbase

使用CREATE EXTERNAL TABLE:

	CREATE EXTERNAL TABLE hbase_table_2(key int, value string)        
	STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'  
	WITH SERDEPROPERTIES ("hbase.columns.mapping" = "cf1:val")  
	TBLPROPERTIES("hbase.table.name" = "some_existing_table");  
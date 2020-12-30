---
title: hive知识整理
date: 2020-12-29 21:16:31
tags:
- hive
categories: 
- 工具
---

# Hive

<!-- toc -->

[TOC]



## 概述

Hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供类SQL查询功能。

本质是将SQL转换为MapReduce程序

主要用途：用来做离线数据分析，比直接用MapReduce开发效率更高

<!--more-->

![img](hive知识整理/timg-1568381886490.jpg) 

数据仓库和数据库的区别

* 数据库是面向事务的设计，数据仓库是面向主题设计的。

* 数据库一般存储业务数据，数据仓库存储的一般是历史数据。

* 数据库设计是尽量避免冗余，一般针对某一业务应用进行设计，比如一张简单的User表，记录用户名、密码等简单数据即可，符合业务应用，但是不符合分析。数据仓库在设计是有意引入冗余，依照分析需求，分析维度、分析指标进行设计。

* 数据库是为捕获数据而设计，数据仓库是为分析数据而设计。

数据仓库分层架构

==源数据层（ODS）==：此层数据无任何更改，直接沿用外围系统数据结构和数据，不对外开放；为临时存储层，是接口数据的临时存储区域，为后一步的数据处理做准备。

==数据仓库层（DW）==：也称为细节层，DW层的数据应该是一致的、准确的、干净的数据，即对源系统数据进行了清洗（去除了杂质）后的数据。

==数据应用层（DA或APP）==：前端应用直接读取的数据源；根据报表、专题分析需求而计算生成的数据。

 

~~~bash
先启动metastore服务再启动hiveserver2服务
/export/servers/hive/bin/beeline
beeline> ! connect jdbc:hive2://hdp3:10000

~~~



hive SQL语句中 select from where group by having order by 的==执行顺序==？

1.from--where--group by--having--select--order by， 

2.from：需要从哪个数据表检索数据 

3.where：过滤表中数据的条件 

4.group by：如何将上面过滤出的数据分组 

5.having：对上面已经分组的数据进行过滤的条件 

6.select：查看结果集中的哪个列，或列的计算结果 

7.order by ：按照什么样的顺序来查看返回的数据

## 1.DDL操作

设置hive程序本地运行模式：

~~~bash
set hive.exec.mode.local.auto=true;
~~~

### 1.1  创建表

```sql
create [external] table [if not exists] tb_name (...) [like] existing_table;
[row format delimited fields terminated by char
					collection items terminated by char
					map keys terminated by char
					lines terminated by char
					...]
[partitioned by ]
[stored as file_format]
[CLUSTERED BY (col_name, col_name, ...) [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]
[LOCATION hdfs_path]


```

1.create table 创建一个指定名字的表，如果表以存在可以用[if not exists]跳过异常

2.[external] 关键字可以让给用户创建一个外部表。

创建内部表时会将数据移动到数据仓库指向的路径，删除表时元数据和数据都被删除。

外部表仅记录数据所在的路径，删除时只删除元数据，不删除数据。

3.[like] 允许用户复制现有的表结构，但是不复制数据

4.[row format delimited] 指定表列与列的分隔符。hive建表的时候默认分隔符是‘\001’,

5.[partitioned by] 分区命令。每个表可以有多个分区，每个分区以文件夹的形式单独存在表文件夹目录下。分区是以字段的形式在表结构中存在。

6.[stored as sequencedile|textfile|refile]   如果文件数据是纯文本，可以使用textfile,如果数据需要压缩，使用sequencedile。

textfile是默认的文件格式，使用delimited子句来读取分隔文件

7.[clustered by (col_name,col_name,....)]   分桶

8.[LOCATION hdfs_path]  指定这张表所在的hdfs上的目录，如果不指定，默认在数据库的目录下面

```sql
create table tb_name as select statement;将sql语句的结果进行保存
create table tb_new like tb_old;创建一张结构与tb_old一样的表
drop table [if exists] tb_name;删除表
truncate table tb_name;清空表
show functions;查看所有的函数
show partitions tb_name;查看所有分区
desc formatted tb_name；查看表信息

```

#### 1.1.1 管理表

最普通的表，默认表的类型就是管理表

```sql
create table tb_name();

```

#### 1.1.2 外部表

```sql
create external table tb_name();

```

特点：在删除时，不会删除表数据

应用场景：1-如果需要多张表共用同一份数据，都建立外部表，使用完以后，删除表互不影响。2-如果数据需要进行额外的使用：存档等等

#### 1.1.3 分区表

```sql
create external table tb_part(
id string
 name string
)
partitioned by (day string)
row format delimited fields terminated by '\t';
--创建分区表，指定分区字段day
load data local inpath '/export/datas/20180718' into table tb_name partition(day='18');
--加载文件数据，创建分区字段day=18
load data local inpath '/export/datas/20180719' into table tb_name partition(day='19');
--加载文件数据，创建分区字段day=19
select * from tb_part where day = 19;
--过滤条件需是分区的字段，如果不是分区字段就会从整个分区目录中查找

```

- 手动分区：加载数据时，手动指定文件的分区

  分区字段为查询的语句的最后一个字段 

  ```sql
  insert overwrite table tb_emp_part partition (deptno)
    select empno
    ename,
    job,
    manager,
    inday,
    salary,
    jiangjin,
    deptno
  from tb_emp_normal;
  
  ```

  - 应用场景：将普通表的数据转换成一个分区表
    		   原始表【tb1】中的数据没有做分区
      		   希望将原始表中的数据按照分区存放到新的分区表[tb2]

  - 自动分区：默认按照原始表的最后一列进行分区

  ```sql
  set hive.exec.dynamic.partition.mode=nonstrict;
  配置自动分区
  show partitions tb_name;
  查看当前所有分区
  
  ```

  数据加载问题：
  1-如果手动将数据文件放入一张普通表的目录下？表能不能读到数据？

  ​	可以，元数据查询时直接将表的目录作为查询目录
  2-如果手动将数据文件放入一张分区表的分区目录下？表能不能读到数据？表的分区能不能读到数据？
  ​	可以的，因为元数据查询时直接将分区的目录作为查询目录
  3-如果手动在HDFS上创建一个分区的目录，将数据放入分区目录下，表能不能读到数据？
  ​	不能，因为Hive中没有该分区的元数据

  方案一：修复表的元数据（资源占用多）

  ```sql
  msck repair table tb_emp;
  
  ```

  方案二：手动向表中添加一个分区

  ```sql
  alter table tb_name add [if exists] partition (dt='20') location '/user/hadoop/dt=20';
  --要求建立的目录必须与分区自动创建的目录格式一样
  
  ```

  删除分区

  ~~~bash
  alter table tb_name drop [if exists] partition (dt='20');
  ~~~

  

  #### 1.1.4  分桶表

  ```sql
  create table tb_emp_bucket(
  empno int ,
  ename string,
  job string,
  manager int,
  inday string,
  salary double,
  jiangjin double,
  deptno int
  ) 
  clustered by (deptno) into 3 BUCKETS
  row format delimited fields terminated by '\t';
  
  ```

  应用场景：大表join大表时

  ```sql
  --开启分桶
  set hive.enforce.bucketing=true;
  
  insert overwrite table tb_emp_bucket
  select * from tb_emp_normal cluster by (deptno);
  
  ```

  连个桶表桶的个数必须相同，或者b表桶是a表的倍数

  ### 1.2 修改表

  增加分区：

  ```sql
  alter table tb_name add partition (dt='20170101') location '/user/hadoop/table_name/dt=20170101';
  
  ```

  删除分区

  ```sql
  alter table tb_name drop if exists partition (dt='20170101')
  
  ```

  修改分区

  ```sql
  alter table tb_name partition (dt='20170101') rename to partition(dt='20170202')
  
  ```

  添加列

  ```sql
  --添加列
  alter table tb_name add|replace columns (col_name string);
  --修改表名
  alter table stu_par rename to stu_par1
  --增加列
  alter table stu1 add columns(score string);
  --修改列类型
  alter table stu1 change column score score double;
  --
  ```

  ## 2.DML操作

  ### 2.1 load 

  在将数据加载到表中时，Hive不会进行任何转换。加载操作是将数据文件移动到与Hive表对应的位置的纯复制/移动操作。

  ```sql
  load data [local] inpath 'filepath' [overwrite] into table tb_name [partition(partcol1=val1,partcol2=val2...)]
  
  ```

  ### 2.2 insert

  Hive中insert主要是结合select查询语句使用，将查询结果插入到表中

  ```sql
  --查询结果的列数要和插入数据表格的列数一致
  insert overwrite table tb_name select statement
  --将查询语句结果保存至HDFS中
  insert overwrite directory "/movie/answer10/" select statement;
  
  ```

  ### 2.3 join

  inner join: 两张表都有结果才有

  left join: 左表有，结果就有

  right join: 右表有，结果就有

  full join：两边任意一边有，结果就有

  cross join:笛卡尔积      --一般用于结果的漏斗计算

  ### 2.4 排序

  ```sql
  set hive.exec.reducers.bytes.per.reducer=<number>
  	每个reduce最多处理多少数据量
  set hive.exec.reducers.max=<number>	
  	最多启动多少个reduce
  set mapreduce.job.reduces=<number>
  	设置reduce的个数
  
  ```

  #### order by

  全局排序，对整体进行排序，只有一个reduce的情况下

  在启用多个reduce的情况下如果使用order by 结果依旧全局有序，但只会启用一个reduce

  ```sql
  select  empno,ename,salary,deptno from tb_emp_normal order by empno;
  
  ```

  #### sort by

  局部排序，多个reduce的 情况下，每个reduce内部有序（分区内有序）

  ```sql
  set mapreduce.job.reduces=3;
  设置reduce个数
  insert overwrite local directory '/export/datas/sort' 
  row format delimited fields terminated by '\t' 
  select  empno,ename,salary,deptno from tb_emp_normal 
  sort by empno;
  
  ```

  #### distribute by

  指定多个reduce情况下，以哪一列作为分区字段。将相同的数据放入同一个结果文件，类似MR中Partition，进行分区，结合sort by使用  

  ```sql
  insert overwrite local directory '/export/datas/dis' 
  row format delimited fields terminated by '\t' 
  select  empno,ename,salary,deptno from tb_emp_normal 
  distribute by deptno 
  sort by empno;
  
  ```

  #### cluster by

  如果sort by与distribute by使用同一个字段可以用此代替，但是排序只能是升序排序，不能指定排序规则为ASC或者DESC。

  

## 3.hive参数配置

### 3.1 Hive shell命令行

针对bin/hive，除了可以当第一代客户端之外。还可以在hive中启动其他用途。

1、 -i  初始化HQL文件。

2、 -e从命令行执行指定的HQL 

3、 -f 执行HQL脚本 

4、 -v 输出执行的HQL语句到控制台 

5、 -p <port> connect to Hive Server on port number 

6、 -hiveconf x=y Use this to set hive/hadoop configuration variables.

例如：

~~~bash
$HIVE_HOME/bin/hive -e 'select * from table a'	

$HIVE_HOME/bin/hive -f /home/my/hive-script.sql

$HIVE_HOME/bin/hive -f hdfs://<namenode>:<port>/hive-script.sql

$HIVE_HOME/bin/hive -i /home/my/hive-init.sql

~~~



### 3.2 Hive 参数配置方式

*Hive参数大全：*

[*https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties*

 开发Hive应用时，不可避免地需要设定Hive的参数。设定Hive的参数可以调优HQL代码的执行效率，或帮助定位问题。然而实践中经常遇到的一个问题是，为什么设定的参数没有起作用？这通常是错误的设定方式导致的。

对于一般参数，有以下三种设定方式：

配置文件   （全局有效）

命令行参数   （对hive启动实例有效）

参数声明   （对hive的连接session有效）

配置文件 

用户自定义配置文件：$HIVE_CONF_DIR/hive-site.xml

默认配置文件：$HIVE_CONF_DIR/hive-default.xml 

用户自定义配置会覆盖默认配置。

另外，Hive也会读入Hadoop的配置，因为Hive是作为Hadoop的客户端启动的，Hive的配置会覆盖Hadoop的配置。

配置文件的设定对本机启动的所有Hive进程都有效。

 命令行参数

启动Hive（客户端或Server方式）时，可以在命令行添加-hiveconf来设定参数	例如：bin/hive -hiveconf hive.root.logger=INFO,console

设定对本次启动的Session（对于Server方式启动，则是所有请求的Sessions）有效。

 参数声明

可以在HQL中使用SET关键字设定参数，这一设定的作用域也是session级的。

比如：

set hive.exec.reducers.bytes.per.reducer=<number>  每个reduce task的平均负载数据量

set hive.exec.reducers.max=<number>   设置reduce task数量的上限

set mapreduce.job.reduces=<number>    指定固定的reduce task数量

但是，这个参数在必要时<业务逻辑决定只能用一个reduce task> hive会忽略

上述三种设定方式的优先级依次递增。即参数声明覆盖命令行参数，命令行参数覆盖配置文件设定。注意某些系统级的参数，例如log4j相关的设定，必须用前两种方式设定，因为那些参数的读取在Session建立以前已经完成了。

## 4.hive中复杂数据类型的使用

### 4.1 数组类型

~~~
--数据如下：vim /export/datas/array.txt
zhangsan	beijing,shanghai,tianjin,hangzhou
wangwu	shanghai,chengdu,wuhan,haerbin
~~~

~~~sql
--创建表
create table complex_array(
name string,
work_locations array<string>
)
row format delimited fields terminated by '\t' --指定列的分隔符
collection items terminated by ',';--指定数组中元素的分隔符

--加载数据
load data local inpath '/export/datas/array.txt' into table complex_array;

--查询
select * from complex_array;
select size(work_locations) from complex_array;
select work_locations[0],work_locations[1] from complex_array;
~~~

### 4.2 map类型

~~~
--数据如下：vim /export/datas/map.txt
1,zhangsan,唱歌:非常喜欢-跳舞:喜欢-游泳:一般般
2,lisi,打游戏:非常喜欢-篮球:不喜欢
~~~

~~~sql
--创建表
create table complex_map(
id int,
name string,
hobby map<string,string>)
row format delimited fields terminated by ',' --指定列的分隔符
collection items terminated by '-' map keys terminated by ':' ;--指定keyvalue的分割

--加载数据
load data local inpath '/export/datas/map.txt' into table complex_map;

--查询
select * from complex_map;
select size(hobby) from complex_map;
select hobby["唱歌"] from complex_map;
~~~

### 4.3 正则类型

```sql
--数据如下:vim /export/datas/regex.txt
tom 男 23 上海
```

```sql
--使用正则加载数据
CREATE TABLE user_regex(
name string,
sex string,
age int,
city string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  "input.regex" = "([^ ]+) ([^ ]+) ([0-9]+) (.+)"
)
STORED AS TEXTFILE;

load data local inpath '/root/regex.txt' into table user_regex;
```

### 4.4 json类型

~~~sql
--通过专门的解析类直接加载一个json格式的数据到Hive中
--数据如下:vim /export/datas/hivedata.json
{"id": 1701439105,"ids": [2154137571,3889177061],"total_number": 493}
{"id": 1701439106,"ids": [2154137571,3889177061],"total_number": 494}
--添加jar包
add jar /export/datas/json-serde-1.3.7-jar-with-dependencies.jar;
~~~

~~~sql
--创建表：
create table tb_json_test2 (
id string,
ids array<string>,
total_number int)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE;


--加载数据
load data local inpath '/export/datas/hivedata.json' into table tb_json_test2;
~~~

### 4.5 python类型

```sql
--创建Python脚本实现将原始表的时间转为对应的星期几
vim /export/datas/weekday_mapper.py

import sys
import datetime

for line in sys.stdin:
  line = line.strip()
  userid, movieid, rating, unixtime = line.split('\t')
  weekday = datetime.datetime.fromtimestamp(float(unixtime)).isoweekday()
  print '\t'.join([userid, movieid, rating, str(weekday)])
 
```

```sql
--加载python脚本并将数据写入新表
add FILE /export/datas/weekday_mapper.py;
INSERT OVERWRITE TABLE u_data_new
SELECT
  TRANSFORM (userid, movieid, rating, unixtime)
  USING 'python weekday_mapper.py'
  AS (userid, movieid, rating, weekday)
FROM u_data;
```

## 5.Hive函数

### 5.1自定义函数

#### 5.1.1 UDF

UDF（User-Defined-Function）普通函数 一进一出

1、自定义一个类，继承UDF,实现一个或重载多个evaluate方法，打包上传jar包到linux环境

```xml
       <!-- 指定该项目可以从哪些地方下载依赖包 -->
			<repository>
				<id>aliyun</id>
				<url>http://maven.aliyun.com/nexus/content/groups/public/</url>
			</repository>
			<repository>
				<id>cloudera</id>
				<url>https://repository.cloudera.com/artifactory/cloudera-repos/</url>
			</repository>
			<repository>
				<id>jboss</id>
				<url>http://repository.jboss.org/nexus/content/groups/public</url>
			</repository>
		</repositories>
		<!--指定字符编码-->
		<properties>
			<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		</properties>
		<!--指定依赖-->
		<dependencies>
			<dependency>
				<groupId>org.apache.hadoop</groupId>
				<artifactId>hadoop-common</artifactId>
				<version>2.6.0-cdh5.14.0</version>
			</dependency>
			<dependency>
				<groupId>org.apache.hadoop</groupId>
				<artifactId>hadoop-hdfs</artifactId>
				<version>2.6.0-cdh5.14.0</version>
			</dependency>

			<dependency>
				<groupId>org.apache.hadoop</groupId>
				<artifactId>hadoop-client</artifactId>
				<version>2.6.0-cdh5.14.0</version>
			</dependency>
			<dependency>
				<groupId>org.apache.hive</groupId>
				<artifactId>hive-exec</artifactId>
				<version>1.1.0-cdh5.14.0</version>
			</dependency>
			<dependency>
				<groupId>org.apache.hive</groupId>
				<artifactId>hive-common</artifactId>
				<version>1.1.0-cdh5.14.0</version>
			</dependency>
			<dependency>
				<groupId>org.apache.hive</groupId>
				<artifactId>hive-cli</artifactId>
				<version>1.1.0-cdh5.14.0</version>
			</dependency>
			<dependency>
				<groupId>org.apache.hive</groupId>
				<artifactId>hive-jdbc</artifactId>
				<version>1.1.0-cdh5.14.0</version>
			</dependency>
		</dependencies>
```

```java
import org.apache.hadoop.hive.ql.exec.UDF;
import org.apache.hadoop.io.Text;
public class UserUDF extends UDF{
	public Text evaluate(Text s){
        if(s==null){
            return null;
        }
        return new Text(s.toString().toLowerCase());
    }
}
```



2、将jar包添加到hive环境中

```
add jar /export/datas/udf.jar;

```

3、在hive中创建一个函数

```
create temporary function fc_name as 'com.neusoft.data.UserUDF';

```

4、使用函数

```
select fc_name(age),name from tb_name;

```

#### 5.1.2自定义udf,udtf,udaf开发

UDF：
1-开发udf程序：继承UDF类，实现一个或者多个evaluate方法
2-打成jar包
3-上传jar包到集群中，并添加到hive的环境变量中，在hive中执行
	add jar /export/datas/udf.jar;
4-创建临时函数：
	create temporary function  transDate as 'cn.itcast.bigdata.hive.TransDate';
5-测试函数：
	select transDate("18/Aug/2019:12:30:05");
6-删除临时函数
	DROP TEMPORARY FUNCTION transDate;

UDTF
1-开发udtf程序：继承UDTF类，重写process方法
2-打成jar包
3-上传jar包到集群中，并添加到hive的环境变量中，在hive中执行
	add jar /export/datas/udtf.jar;
4-创建临时函数：
	create temporary function  transMap as 'cn.itcast.bigdata.hive.UserUDTF';
5-测试函数：
	第一种用法：直接调用
		select transMap("uuid=root&url=www.taobao.com") as (userCol1,userCol2);
	第二种用法：和侧视图一起使用
		select deptno,deptname,a.* from tb_dept lateral view transMap("uuid=root&url=www.taobao.com") a as col1,col2;
	注意：
		1-udtf只能直接select中使用
		2-不可以添加其他字段使用
		3-不可以嵌套调用
		4-不可以和group by/cluster by/distribute by/sort by一起使用





UDAF
1-开发udaf程序：继承UDAF类，重写iterate方法
2-打成jar包
3-上传jar包到集群中，并添加到hive的环境变量中，在hive中执行
	add jar /export/datas/udaf.jar;
4-创建临时函数：
	create temporary function  userMax as 'cn.itcast.bigdata.hive.UserUDAF';
5-测试函数：
	select userMax(deptno) from tb_dept;

### 5.2UDAF

UDAF（User-Defined Aggregation Function）聚合函数，多进一出

#### 窗口函数：SUM、AVG、COUNT、MAX、MIN

功能：用于实现数据分区后的聚合
	-》语法：fun_name(col1) over (partition by col2 order by col3)
				实现功能	over  按照什么分区，分区内部按照什么排序
	-》示例：实现分区内的累加，其他的原理类似

~~~
cookie1,2018-04-10,1
cookie1,2018-04-11,5
cookie1,2018-04-12,7
cookie1,2018-04-13,3
cookie2,2018-04-13,3
cookie2,2018-04-14,2
cookie2,2018-04-15,4
cookie1,2018-04-14,2
cookie1,2018-04-15,4
cookie1,2018-04-16,4
cookie2,2018-04-10,1
cookie2,2018-04-11,5
cookie2,2018-04-12,7
~~~

~~~sql
--创建表
create database db_function;
use db_function;
create table user_f1(
cookieid string,
daystr string,
pv int
) row format delimited fields terminated by ',';
--加载数据
load data local inpath '/export/datas/window.txt' into table user_f1;
set hive.exec.mode.local.auto=true;
--实现分区内起始到当前行的pv累加，默认窗口：取第一行开始到当前行的和
select 
  cookieid,
  daystr,
  pv,
  sum(pv) over(partition by cookieid order by daystr) as pv1 
from 
  user_f1;
   
--实现分区内所有pv的累加，不指定排序默认窗口：从第一行到最后一行
select 
  cookieid,
  daystr,
  pv,
  sum(pv) over(partition by cookieid ) as pv2
from 
  user_f1;
  
    
--手动指定窗口的大小：分区
rows between 起始位置 and 结束位置
rows between unbounded preceding and current row

--实现分区内起始到当前行的pv累加
select 
  cookieid,
  daystr,
  pv,
  sum(pv) over(partition by cookieid order by daystr rows between unbounded preceding and current row) as pv3
from 
  user_f1;
--实现分区内指定前N行到当前行的pv累加
select 
  cookieid,
  daystr,
  pv,
  sum(pv) over(partition by cookieid order by daystr rows between 3 preceding and current row) as pv4
from 
  user_f1;
--实现分区内指定前N行到后N行的pv累加 
select 
  cookieid,
  daystr,
  pv,
  sum(pv) over(partition by cookieid order by daystr rows between 3 preceding and 1 following) as pv5
from 
  user_f1;
--实现分区内指定当前行到后N行的pv累加   
select 
  cookieid,
  daystr,
  pv,
  sum(pv) over(partition by cookieid order by daystr rows between current row and unbounded following) as pv6
from 
  user_f1;
  
  
- preceding：往前
- following：往后
- current row：当前行
- unbounded：起点
- unbounded preceding 表示从前面的起点
- unbounded following：表示到后面的终点
~~~

#### 窗口函数：LAG、LEAD、FIRST_VALUE、LAST_VALUE

FIRST_VALUE
	功能：取每个分区内某列的第一个值
	语法：FIRST_VALUE(col) over (partition by col1 order by col2)

~~~sql
--取每个部门薪资最高的员工编号
select
  empno,
  ename,
  salary,
  deptno,
  FIRST_VALUE(ename) over (partition by deptno order by salary desc) as first
from
  db_emp.tb_emp_normal;
~~~

LAST_VALUE
	功能：取每个分区内某列的最后一个值
	语法：LAST_VALUE() over (partition by col1 order by col2)
	注意：默认窗口是从第一条到当前条

~~~sql
--取每个部门薪资最低的员工编号
select
  empno,
  ename,
  salary,
  deptno,
  LAST_VALUE(empno) over (partition by deptno order by salary desc) as last
from
  db_emp.tb_emp_normal; 

select
  empno,
  ename,
  salary,
  deptno,
  LAST_VALUE(empno) over (partition by deptno order by salary desc rows between unbounded preceding and unbounded following) as last
from
  db_emp.tb_emp_normal;
~~~

LAG
	功能：取每个分区内某列的前N个值
	语法：LAG(col,N,defaultValue) over (partition by col1 order by col2)

~~~sql
select
  empno,
  ename,
  salary,
  deptno,
  LAG(salary,1,0) over (partition by deptno order by salary) as deptno2
from
  db_emp.tb_emp_normal;
~~~

LEAD
	功能：向后取每个分区内某列的前N个值
	语法：LEAD(col,N,defaultValue) over (partition by col1 order by col2)

~~~sql
select
  empno,
  ename,
  salary,
  deptno,
  LEAD(salary,1,0) over (partition by deptno order by salary) as deptno2
from
  db_emp.tb_emp_normal;
~~~





#### 分析函数：ROW_NUMBER、RANK、DENSE_RANK、NTILE

==ROW_NUMBER==
	功能：用于实现分区内记录编号
	语法：row_number() over (partition by col1 order by col2)

~~~sql
--统计每个部门薪资最高的前两名
select * from 
(select
  empno,
  ename,
  salary,
  deptno,
  row_number() over (partition by deptno order by salary desc) as numb
from
  db_emp.tb_emp_normal) t where t.numb < 3;
~~~

RANK
	功能：用于实现分区内排名编号[会留空位]
		与row_number的区别：
			row_number：如果排序时数值相同，继续编号
			rank：如果排序时数值相同，编号不变，但留下空位
	语法：rank() over (partition by col1 order by col2)

~~~sql
--统计每个部门薪资排名
select
  empno,
  ename,
  salary,
  deptno,
  rank() over (partition by deptno order by salary desc) as numb
from
  db_emp.tb_emp_normal;
~~~

DENSE_RANK
	功能：用于实现分区内排名编号[不留空位]
		与rank的区别：
			==row_number：如果排序时数值相同，编号不变，并留下排名空位==
			==dense_rank：如果排序时数值相同，编号不变，不留空位==
			==rank：如果排序时数值相同，编号不变，但留下空位==
	语法：dense_rank() over (partition by col1 order by col2)

~~~sql
--统计每个部门薪资排名
select
  empno,
  ename,
  salary,
  deptno,
  dense_rank() over (partition by deptno order by salary desc) as numb
from
  db_emp.tb_emp_normal;
~~~

NTILE
	功能：将每个分区内排序后的结果均分成N份【如果不能均分，优先分配编号小的】
		本质：将每个分区拆分成更小的分区
	语法：NTILE(N) over (partition by col1 order by col2)

~~~sql
--统计每个部门薪资排名，将每个部门的薪资分为两个部分，区分高薪和低薪
select
  empno,
  ename,
  salary,
  deptno,
  NTILE(2) over (partition by deptno order by salary desc) as numb
from
  db_emp.tb_emp_normal;
~~~



### 5.3UDTF

UDTF（User-Defined Table-Generating Functions）表生成函数 一进多出

#### lateral view

分类：视图
功能：配合UDTF来使用,把某一行数据拆分成多行数据
	与UDTF直接使用的区别：
		==很多的UDTF不能将结果与源表进行关联，使用lateral view==
		可以将UDTF拆分的单个字段数据与原始表数据关联上==
使用方式：
	tabel A lateral view UDTF(xxx) 视图名 as a,b,c

~~~sql
--准备数据:vim /export/datas/lateral.txt
1	http://facebook.com/path/p1.php?query=1
2	http://www.baidu.com/news/index.jsp?uuid=frank
3	http://www.jd.com/index?source=baidu

--创建表
create table tb_url(
id int,
url string
) row format delimited fields terminated by '\t';

--加载数据
load data local inpath '/export/datas/lateral.txt' into table tb_url;

--使用UDTF解析
SELECT parse_url_tuple(url, 'HOST') from tb_url;

--使用UDTF+lateral view
select a.id,b.* from tb_url a lateral view parse_url_tuple(url, 'HOST') b as host;

--对比
SELECT id,parse_url_tuple(url, 'HOST') from tb_url;--失败，UDTF函数不能与字段连用
~~~



#### explode

功能：函数可以将一个array或者map展开
	explode(array)：
		将array列表里的每个元素生成一行
	explode(map)：
		每一对元素作为一行，key为一列，value为一列
使用方式：
	1-直接使用
	2-与lateral view连用

~~~sql
--实现wordcount【explode(array)】
	select explode(split(word," ")) from wc1;
--将兴趣爱好展开【explode(map)】
	select explode(hobby) from complex_map;
--与侧视图连用
	select a.name,b.* from complex_map a lateral view explode(hobby) b as hobby,deep;
~~~

#### reflect

功能：用于在Hive中直接调用Java中静态类的方法
	用法：reflect(classname,method,args)

~~~sql
select reflect("java.util.UUID", "randomUUID");
select reflect("java.lang.Math","max",20,30);
select reflect("org.apache.commons.lang.math.NumberUtils","isNumber","123");
~~~

#### get_json_object

处理json格式数据

~~~sql
--创建数据：vim /export/datas/hivedata.json
{"id": 1701439105,"ids": [2154137571,3889177061],"total_number": 493}
{"id": 1701439106,"ids": [2154137571,3889177061],"total_number": 494}
--创建表：
create table tb_json_test1 (
json string
);
--加载数据
load data local inpath '/export/datas/hivedata.json' into table tb_json_test1;
~~~

~~~sql
--处理读取
select 
  get_json_object(t.json,'$.id'), 
  get_json_object(t.json,'$.total_number') 
from 
  tb_json_test1 t ;
  
select 
  t2.* 
from 
  tb_json_test1 t1 
lateral view 
  json_tuple(t1.json, 'id', 'total_number') t2 as c1,c2;
~~~



#### COALESCE

COALESCE(col,0)

如果单列值为null，替换为默认值0

concat

~~~sql
concat( 'liubei','xihuan','xiaoqiao' )
liubeixihuanxiaoqiao
~~~



#### concat_ws&collect_set

concat_ws('|', collect_set(c_id))

~~~sql
id    name
1001    A
1001    B
1001    C
-------------------------
id      name
1001    A,B,C

select id,collect_list(name) from tb_ss group by id

如果需要去重课可以使用collect_set,返回的是数组
concat_ws('|',collect_set(c_id))可以将数组内容按|拼接

如果该列不是string，先用cast(col) as string 转换为string类型
select id,concat_ws(',',collect_list(cast (name as string))) from tb_ss group by id
~~~



#### instr

查找字符串str中子字符串substr的位置， 如果查找失败将返回0，如果任一参数为Null将返回null，注意位置为从1开始的 

~~~
instr(string str, string substr)
~~~



#### substring

截取字符串

~~~
hello
substring(col,1,2) -> 'he'
substring(col,-2,2) -> 'lo'
~~~



### 5.4常见自带的hive函数

show functions;

使用desc function  extended 函数名查看帮助

•UNIX时间戳转日期函数: from_unixtime

• 获取当前UNIX时间戳函数: unix_timestamp

•日期转UNIX时间戳函数: unix_timestamp

• 指定格式日期转UNIX时间戳函数: unix_timestamp

•日期时间转日期函数: to_date

•日期转年函数: year

• 日期转月函数: month

• 日期转天函数: day

• 日期转小时函数: hour

• 日期转分钟函数: minute

• 日期转秒函数: second

• 日期转周函数: weekofyear

• 两个日期之间有多少个月:months_between('2020-01-10', u.birthday)(多用户得出用户年龄)

• 日期比较函数: datediff

• 日期增加函数: date_add

• 日期减少函数: date_sub

• 取当前天的下一周的周几：next_day("xxxx-xx-xx","Mo")

• 取当前月的最后一天：last_day("xxxx-xx-xx")

•If函数: if

•非空查找函数: COALESCE

•条件判断函数：CASE

•字符串长度函数：length

•字符串反转函数：reverse

•字符串连接函数：concat

• 带分隔符字符串连接函数：concat_ws

• 字符串截取函数：substr,substring

•正则表达式替换函数：regexp_replace

•正则表达式解析函数：regexp_extract

•URL解析函数：parse_url

​							parse_url_tuple

•json解析函数：get_json_object

​							json_tuple

•分割字符串函数: split

•集合查找函数: find_in_set

### 5.5关于union和union all

总结分析

1. 子查询相当于表名，使用 from 关键字需要指定真实表名或表别名。

2. hive 不支持union ，只支持union all 

3. 子查询中使用union all 时，在子查询里不能使用count、sum 等 聚合函数 

4. 两表直接进行union all 可以使用count、sum 等聚合函数 

5. 两张表进行union all 取相同的字段名称，可正常输出指定数据内容，且结果为两张表的结果集

## 6.hive语法要点

~~~sql
(1).Hive不支持join的非等值连接,不支持or
分别举例如下及实现解决办法。
  不支持不等值连接
       错误:select * from a inner join b on a.id<>b.id
       替代方法:select * from a inner join b on a.id=b.id and a.id is null;
 不支持or
       错误:select * from a inner join b on a.id=b.id or a.name=b.name
       替代方法:select * from a inner join b on a.id=b.id
                union all
                select * from a inner join b on a.name=b.name
  两个sql union all的字段名必须一样或者列别名要一样。
        
(2).分号字符:不能智能识别concat(‘;’,key)，只会将‘；’当做SQL结束符号。
    •分号是SQL语句结束标记，在HiveQL中也是，但是在HiveQL中，对分号的识别没有那么智慧，例如：
        •select concat(key,concat(';',key)) from dual;
    •但HiveQL在解析语句时提示：
        FAILED: Parse Error: line 0:-1 mismatched input '<EOF>' expecting ) in function specification
    •解决的办法是，使用分号的八进制的ASCII码进行转义，那么上述语句应写成：
        •select concat(key,concat('\073',key)) from dual;

(3).不支持INSERT INTO 表 Values（）, UPDATE, DELETE等操作.这样的话，就不要很复杂的锁机制来读写数据。
    INSERT INTO syntax is only available starting in version 0.8。INSERT INTO就是在表或分区中追加数据。

(4).HiveQL中String类型的字段若是空(empty)字符串, 即长度为0, 那么对它进行IS NULL的判断结果是False，使用left join可以进行筛选行。

(5).不支持 ‘< dt <’这种格式的范围查找，可以用dt in(”,”)或者between替代。

(6).Hive不支持将数据插入现有的表或分区中，仅支持覆盖重写整个表，示例如下：
    INSERT OVERWRITE TABLE t1 SELECT * FROM t2;
    
(7).group by的字段,必须是select后面的字段，select后面的字段不能比group by的字段多.
    如果select后面有聚合函数,则该select语句中必须有group by语句
    而且group by后面不能使用别名
    
(8).hive的0.13版之前select , where 及 having 之后不能跟子查询语句(一般使用left join、right join 或者inner join替代)

(9).先join(及inner join) 然后left join或right join

(10).hive不支持group_concat方法,可用 concat_ws('|', collect_set(str)) 实现

(11).not in 后不能包含查询语句,可用left join tmp on tableName.id = tmp.id where tmp.id is null 替代实现

1.case when ... then ... else ... end

2.length(string)

3.cast(string as bigint)

4.rand()       返回一个0到1范围内的随机数

5.ceiling(double)    向上取整

6.substr(string A, int start, int len)

7.collect_set(col)函数只接受基本数据类型，它的主要作用是将某字段的值进行去重汇总，产生array类型字段

8.concat()函数
    1、功能：将多个字符串连接成一个字符串。
    2、语法：concat(str1, str2,...)
    返回结果为连接参数产生的字符串，如果有任何一个参数为null，则返回值为null。

    9.concat_ws()函数
    1、功能：和concat()一样，将多个字符串连接成一个字符串，但是可以一次性指定分隔符～（concat_ws就是concat with separator）
    2、语法：concat_ws(separator, str1, str2, ...)
    说明：第一个参数指定分隔符。需要注意的是分隔符不能为null，如果为null，则返回结果为null。

    10.nvl(expr1, expr2)：空值转换函数  nvl(x,y)    Returns y if x is null else return x

11.if(boolean testCondition, T valueTrue, T valueFalse)

12.row_number()over()分组排序功能,over()里头的分组以及排序的执行晚于 where group by  order by 的执行。

13.获取年、月、日、小时、分钟、秒、当年第几周
    select 
        year('2018-02-27 10:00:00')       as year
        ,month('2018-02-27 10:00:00')      as month
        ,day('2018-02-27 10:00:00')        as day
        ,hour('2018-02-27 10:00:00')       as hour
        ,minute('2018-02-27 10:00:00')     as minute
        ,second('2018-02-27 10:00:00')     as second
        ,weekofyear('2018-02-27 10:00:00') as weekofyear
  获取当前时间:
        1).current_timestamp
        2).unix_timestamp()
        3).from_unixtime(unix_timestamp())
        4).CURRENT_DATE
~~~



## 7.hive优化



### 7.1大表join大表优化

```sql
如果Hive优化实战2中mapjoin中小表dim_seller很大呢？比如超过了1GB大小？这种就是大表join大表的问题。首先引入一个具体的问题场景，然后基于此介绍各自优化方案。

1、问题场景
问题场景如下：

A表为一个汇总表，汇总的是卖家买家最近N天交易汇总信息，即对于每个卖家最近N天，其每个买家共成交了多少单，总金额是多少，假设N取90天，汇总值仅取成交单数。

A表的字段有：buyer_id、seller_id、pay_cnt_90day。

B表为卖家基本信息表，其字段有seller_id、sale_level，其中sale_levels是卖家的一个分层评级信息，比如吧卖家分为6个级别：S0、S1、S2、S3、S4和S5。

要获得的结果是每个买家在各个级别的卖家的成交比例信息，比如：

某买家：S0:10%；S1:20%；S2:20%；S3:10%；S4:20%；S5:10%。

正如mapjoin中的例子一样，第一反应是直接join两表并统计：

select

m.buyer_id,

sum(pay_cnt_90day)  as pay_cnt_90day,

sum(case when m.sale_level = 0  then pay_cnt_90day  end)  as pay_cnt_90day_s0,

sum(case when m.sale_level = 1  then pay_cnt_90day  end)  as pay_cnt_90day_s1,

sum(case when m.sale_level = 2  then pay_cnt_90day  end)  as pay_cnt_90day_s2,

sum(case when m.sale_level = 3  then pay_cnt_90day  end)  as pay_cnt_90day_s3,

sum(case when m.sale_level = 4  then pay_cnt_90day  end)  as pay_cnt_90day_s4,

sum(case when m.sale_level = 5  then pay_cnt_90day  end)  as pay_cnt_90day_s5

from (

select  a.buer_id,  a.seller_id,  b.sale_level, a.pay_cnt_90day

from (  select buyer_id,  seller_id,  pay_cnt_90day   from table_A)  a

join

(select seller_id,  sale_level  from table_B)  b

on  a.seller_id  = b.seller_id

)  m

group by m.buyer_id

但是此SQL会引起数据倾斜，原因在于卖家的二八准则，某些卖家90天内会有几百万甚至上千万的买家，但是大部分的卖家90天内买家的数目并不多，join table_A和table_B的时候，

ODPS会按照seller_id进行分发，table_A的大卖家引起了数据倾斜。

但是数据本身无法用mapjoin table_B解决，因为卖家超过千万条，文件大小有几个GB，超过了1GB的限制。

优化方案1：转为mapjoin
一个很正常的想法是，尽管B表无法直接mapjoin, 但是是否可以间接mapjoin它呢？

实际上此思路有两种途径：限制行和限制列。

限制行的思路是不需要join B全表，而只需要join其在A表中存在的，对于本问题场景，就是过滤掉90天内没有成交的卖家。

限制列的思路是只取需要的字段。

加上如上的限制后，检查过滤后的B表是否满足了Hive  mapjoin的条件，如果能满足，那么添加过滤条件生成一个临时B表，然后mapjoin该表即可。采用此思路的语句如下：



select

m.buyer_id,

sum(pay_cnt_90day)  as pay_cnt_90day,

sum(case when m.sale_level = 0  then pay_cnt_90day  end)  as pay_cnt_90day_s0,

sum(case when m.sale_level = 1  then pay_cnt_90day  end)  as pay_cnt_90day_s1,

sum(case when m.sale_level = 2  then pay_cnt_90day  end)  as pay_cnt_90day_s2,

sum(case when m.sale_level = 3  then pay_cnt_90day  end)  as pay_cnt_90day_s3,

sum(case when m.sale_level = 4  then pay_cnt_90day  end)  as pay_cnt_90day_s4,

sum(case when m.sale_level = 5  then pay_cnt_90day  end)  as pay_cnt_90day_s5

from ( 

select  /*+mapjoin(b)*/

a.buer_id,  a.seller_id,  b.sale_level, a.pay_cnt_90day

from (  select buyer_id,  seller_id,  pay_cnt_90day   from table_A)  a

join

(

select seller_id,  sale_level  from table_B b0

join 

(select seller_id from table_A group by seller_id) a0

on b0.seller_id = a0.selller_id

)  b

on  a.seller_id  = b.seller_id

)  m

group by m.buyer_id

此方案在一些情况可以起作用，但是很多时候还是无法解决上述问题，因为大部分卖家尽管90天内买家不多，但还是有一些的，过滤后的B表仍然很多。


优化方案2：join时用case when语句
此种解决方案应用场景是：倾斜的值是明确的而且数量很少，比如null值引起的倾斜。其核心是将这些引起倾斜的值随机分发到Reduce,其主要核心逻辑在于join时对这些特殊值concat随机数，

从而达到随机分发的目的。此方案的核心逻辑如下：

select a.user_id, a.order_id, b.user_id

from table_a a join table_b b

on (case when a.user_is is null then concat('hive', rand()) else a.user_id end) = b.user_id

Hive 已对此进行了优化，只需要设置参数skewinfo和skewjoin参数，不修改SQL代码，例如，由于table_B的值“0” 和“1”引起了倾斜，值需要做如下设置：

set hive.optimize.skewinfo=table_B:(selleer_id) [ ( "0") ("1") ) ] 

set hive.optimize.skewjoin = true;

但是方案2因为无法解决本问题场景的倾斜问题，因为倾斜的卖家大量存在而且动态变化。


优化方案3：倍数B表，再取模join
1、通用方案
此方案的思路是建立一个numbers表，其值只有一列int 行，比如从1到10（具体值可根据倾斜程度确定），然后放大B表10倍，再取模join。代码如下：



select

m.buyer_id,

sum(pay_cnt_90day)  as pay_cnt_90day,

sum(case when m.sale_level = 0  then pay_cnt_90day  end)  as pay_cnt_90day_s0,

sum(case when m.sale_level = 1  then pay_cnt_90day  end)  as pay_cnt_90day_s1,

sum(case when m.sale_level = 2  then pay_cnt_90day  end)  as pay_cnt_90day_s2,

sum(case when m.sale_level = 3  then pay_cnt_90day  end)  as pay_cnt_90day_s3,

sum(case when m.sale_level = 4  then pay_cnt_90day  end)  as pay_cnt_90day_s4,

sum(case when m.sale_level = 5  then pay_cnt_90day  end)  as pay_cnt_90day_s5

from (

select  a.buer_id,  a.seller_id,  b.sale_level, a.pay_cnt_90day

from (  select buyer_id,  seller_id,  pay_cnt_90day   from table_A)  a

join

(

select  /*+mapjoin(members)*/

seller_id,  sale_level ,member

from table_B

join members

)  b

on  a.seller_id  = b.seller_id

and mod(a.pay_cnt_90day,10)+1 = b.number 

)  m

group by m.buyer_id

此思路的核心在于，既然按照seller_id分发会倾斜，那么再人工增加一列进行分发，这样之前倾斜的值的倾斜程度会减少到原来的1/10，可以通过配置numbers表改放大倍数来降低倾斜程度，

但这样做的一个弊端是B表也会膨胀N倍。

2、专用方案
通用方案的思路把B表的每条数据都放大了相同的倍数，实际上这是不需要的，只需要把大卖家放大倍数即可：需要首先知道大卖家的名单，即先建立一个临时表动态存放每天最新的大卖家（

比如dim_big_seller）,同时此表的大卖家要膨胀预先设定的倍数（1000倍）。

在A表和B表分别新建一个join列，其逻辑为：如果是大卖家，那么concat一个随机分配正整数（0到预定义的倍数之间，本例为0~1000）；如果不是，保持不变。具体代码如下：



select

m.buyer_id,

sum(pay_cnt_90day)  as pay_cnt_90day,

sum(case when m.sale_level = 0  then pay_cnt_90day  end)  as pay_cnt_90day_s0,

sum(case when m.sale_level = 1  then pay_cnt_90day  end)  as pay_cnt_90day_s1,

sum(case when m.sale_level = 2  then pay_cnt_90day  end)  as pay_cnt_90day_s2,

sum(case when m.sale_level = 3  then pay_cnt_90day  end)  as pay_cnt_90day_s3,

sum(case when m.sale_level = 4  then pay_cnt_90day  end)  as pay_cnt_90day_s4,

sum(case when m.sale_level = 5  then pay_cnt_90day  end)  as pay_cnt_90day_s5

from (

select  a.buer_id,  a.seller_id,  b.sale_level, a.pay_cnt_90day

from (  

select  /*+mapjoin(big)*/

buyer_id,  seller_id,  pay_cnt_90day,

if(big.seller_id is not null, concat(  table_A.seller_id,  'rnd',  cast(  rand() * 1000 as bigint ), table_A.seller_id)  as seller_id_joinkey

from table_A

left outer join

--big表seller_id有重复，请注意一定要group by 后再join,保证table_A的行数保持不变

（select seller_id  from dim_big_seller  group by seller_id）big

on table_A.seller_id = big.seller_id

)  a

join

(

select  /*+mapjoin(big)*/

seller_id,  sale_level ,

--big表的seller_id_joinkey生成逻辑和上面的生成逻辑一样

coalesce(seller_id_joinkey,table_B.seller_id) as seller_id_joinkey

from table_B

left out join

--table_B表join大卖家表后大卖家行数扩大1000倍，其它卖家行数保持不变

(select seller_id, seller_id_joinkey from dim_big_seller) big

on table_B.seller_id= big.seller_id

)  b

on  a.seller_id_joinkey= b.seller_id_joinkey

and mod(a.pay_cnt_90day,10)+1 = b.number 

)  m

group by m.buyer_id

相比通用方案，专用方案的运行效率明细好了许多，因为只是将B表中大卖家的行数放大了1000倍，其它卖家的行数保持不变，但同时代码复杂了很多，而且必须首先建立大数据表。

方案4：动态一分为二
实际上方案2和3都用了一分为二的思想，但是都不彻底，对于mapjoin不能解决的问题，终极解决方案是动态一分为二，即对倾斜的键值和不倾斜的键值分开处理，不倾斜的正常join即可，倾斜的把他们找出来做mapjoin，最后union all其结果即可。

但是此种解决方案比较麻烦，代码复杂而且需要一个临时表存放倾斜的键值。代码如下：

--由于数据倾斜，先找出90天买家超过10000的卖家

insert overwrite table  temp_table_B

select 

m.seller_id,  n.sale_level

from (

select   seller_id

from (

select seller_id,count(buyer_id) as byr_cnt

from table_A

group by seller_id

) a

where a.byr_cnt >10000

) m

left join 

(

select seller_id, sale_level  from table_B

) n

on m.seller_id = n.seller_id;



--对于90天买家超过10000的卖家直接mapjoin,对其它卖家直接正常join即可。



select

m.buyer_id,

sum(pay_cnt_90day)  as pay_cnt_90day,

sum(case when m.sale_level = 0  then pay_cnt_90day  end)  as pay_cnt_90day_s0,

sum(case when m.sale_level = 1  then pay_cnt_90day  end)  as pay_cnt_90day_s1,

sum(case when m.sale_level = 2  then pay_cnt_90day  end)  as pay_cnt_90day_s2,

sum(case when m.sale_level = 3  then pay_cnt_90day  end)  as pay_cnt_90day_s3,

sum(case when m.sale_level = 4  then pay_cnt_90day  end)  as pay_cnt_90day_s4,

sum(case when m.sale_level = 5  then pay_cnt_90day  end)  as pay_cnt_90day_s5

from (

select  a.buer_id,  a.seller_id,  b.sale_level, a.pay_cnt_90day

from (  select buyer_id,  seller_id,  pay_cnt_90day   from table_A)  a

join

(

select seller_id,  a.sale_level 

from table_A  a

left join temp_table_B b

on a.seller_id = b.seller_id

where b.seller_id is not null

)  b

on  a.seller_id  = b.seller_id

union all



select  /*+mapjoin(b)*/

a.buer_id,  a.seller_id,  b.sale_level, a.pay_cnt_90day

from ( 

select buyer_id,  seller_id,  pay_cnt_90day   

from table_A

)  a

join

(

select seller_id,  sale_level  from table_B 

)  b

on  a.seller_id  = b.seller_id

)  m  group by m.buyer_id

) m

group by m.buyer_id



总结：方案1、2以及方案3中的同用方案不能保证解决大表join大表问题，因为它们都存在种种不同的限制和特定使用场景。

而方案3的专用方案和方案4是推荐的优化方案，但是它们都需要新建一个临时表来存储每日动态变化的大卖家。相对方案4来说，方案3的专用方案不需要对代码框架进行修改，但是B表会被放大，所以一定要是是维度表，不然统计结果会是错误的。方案4最通用，自由度最高，但是对代码的更改也最大，甚至修改更难代码框架，可以作为终极方案使用。
```
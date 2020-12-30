---
title: dataworks命令
date: 2020-12-24 22:27:32
tags:
- dataworks
- 随笔
categories:
 - 工具
---

<!-- toc -->

[TOC]

# 一、DDL语句

```sql

alter table table_name1 rename to table_name2;   -- 修改表名
alter table table_name1 changeowner to 'ALIYUN$xxx@aliyun.com'; --修改表的所有人
alter table table_name1 set COMMENT 'tbl comment'; --修改表注释
ALTER TABLE sale_detail CHANGE COLUMN customer_name RENAME TO customer; --修改字段名 
ALTER TABLE sale_detail CHANGE COLUMN customer COMMENT 'customer';--修改字段注释
ALTER TABLE table_name CHANGE COLUMN old_col_name new_col_name column_type COMMENT 'column_comment';--同时修改列名和注释
ALTER TABLE table_name ADD COLUMNS (col_name1 type1 comment 'XXX',col_name2 type2 comment 'XXX');--添加列和注释

```


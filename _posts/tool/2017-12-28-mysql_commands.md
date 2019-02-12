---
layout: post
title: mysql命令记录
category: 工具
tags: [Mysql命令记录]
keywords: mysql, sql
---

### 链接:

MySQL Online DDL： [https://www.cnblogs.com/mysql-dba/p/6192897.html](https://www.cnblogs.com/mysql-dba/p/6192897.html)

### 外键约束
#### 禁用外键约束:
```
SET FOREIGN_KEY_CHECKS=0
```
#### 启动外键约束:
```
SET FOREIGN_KEY_CHECKS=1;
```
#### 查看当前FOREIGN_KEY_CHECKS的值可用如下命令:
```
SELECT  @@FOREIGN_KEY_CHECKS;
```

### 外键
#### 添加外键
```
ALTER TABLE `wenshu_data`.`lawyersintelprop` ADD CONSTRAINT `fk_lawyersintelprop__wenshu_id` FOREIGN KEY (`wenshu_id`) REFERENCES `wenshu_data`.`basicintelprop` (`wenshu_id`);
```
#### 删除外键
```
ALTER TABLE `wenshu_data`.`lawyerscomsecinsnot` DROP FOREIGN KEY `fk_lawyerscomsecinsnot__wenshu_id`;
```

### 自增id
#### 主键自增id
```
MySQL [data_third_part]> alter table wenshu_info005 add id int;
Query OK, 0 rows affected (6 min 35.09 sec)
Records: 0  Duplicates: 0  Warnings: 0

MySQL [data_third_part]> alter table wenshu_info005 drop primary key;
Query OK, 129945 rows affected (3 min 18.66 sec)
Records: 129945  Duplicates: 0  Warnings: 0

MySQL [data_third_part]> alter table wenshu_info005 change id id int not null auto_increment primary key;
Query OK, 129945 rows affected (3 min 19.04 sec)
Records: 129945  Duplicates: 0  Warnings: 0
```
#### 非主键自增id
```
alter table basicintelprop add id int auto_increment Unique;
```

### 索引：
#### 普通增加索引
```
索引的通常命名： idx_<表名字>__<添加索引字段1>__<添加索引字段2>
CREATE INDEX `idx_cnipsundata__agent` ON `cnipsundata` (`agent`)
```
#### 删除索引
```
ALTER TABLE `wenshu_data`.`lawyerinfo` DROP INDEX `idx_lawyerinfo__name`;
```
#### 修改复合索引
```
ALTER TABLE `wenshu_data`.`lawyerinfo` DROP INDEX `idx_lawyerinfo__name__lawyer_firm__region`, ADD INDEX `idx_lawyerinfo__name__lawyer_firm__region1` USING BTREE (`name`, `lawyer_firm`, `region`) comment '';
```
![composite-index](/assets/img/tool/mysql/composite-index.png)

### 大表结构修改
#### 添加字段：
```
1600万数据(51GB存储),表结构较大,添加字段, 用3小时40分钟：
mysql> select id from cnipsundata order by id desc limit 1;
+----------+
| id       |
+----------+
| 16709353 |
+----------+
1 row in set (0.05 sec)
mysql> alter table cnipsundata add province varchar(64);
Query OK, 0 rows affected (3 hours 43 min 10.59 sec)
Records: 0  Duplicates: 0  Warnings: 0

```

### 查看MySQL数据库大小
#### 查看所有数据库大小
```
select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from INFORMATION_SCHEMA.TABLES;
```
#### 查看指定数据库大小
```
select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from INFORMATION_SCHEMA.TABLES where table_schema='CarData';
```
#### 查看指定数据库的指定表的大小
```
select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from INFORMATION_SCHEMA.TABLES where table_schema='CarData' and table_name='driver020294';
```
#### 查看指定数据库指定表的其他大小
```
select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data_size,
    -> concat(round(sum(MAX_DATA_LENGTH/1024/1024),2),'MB') as max_data_size,
    -> concat(round(sum(INDEX_LENGTH/1024/1024),2),'MB') as index_size,
    -> concat(round(sum(DATA_FREE/1024/1024),2),'MB') as data_free
    -> from INFORMATION_SCHEMA.TABLES where table_schema='CarData' and table_name='driver020294';
```

#### 插入更新语句ON DUPLICATE KEY UPDATE
- if-else写法(效率太差，每次都需要执行两条SQL语句; 高并发的情况下数据会出问题，不能保证原子性):
```
if not exists (select node_name from node_status where node_name = target_name)
      insert into node_status(node_name,ip,...) values('target_name','ip',...)
else
      update node_status set ip = 'ip',site = 'site',... where node_name = target_name
```
- ON DUPLICATE KEY UPDATE:

如果你插入的记录导致一个UNIQUE索引或者primary key(主键)出现重复，那么就会认为该条记录存在，则执行update语句而不是insert语句，反之，则执行insert语句而不是更新语句。

```
INSERT INTO tablename(field1,field2, field3, ...) VALUES(value1, value2, value3, ...) ON DUPLICATE KEY UPDATE field1=value1,field2=value2, field3=value3, ...;
```


---
layout: post
title: mysql命令记录
category: 数据库
tags: [Mysql命令记录]
keywords: mysql, sql
---

## 链接:

MySQL Online DDL： [https://www.cnblogs.com/mysql-dba/p/6192897.html](https://www.cnblogs.com/mysql-dba/p/6192897.html)

## 分区键、主键和唯一键(Partitioning Keys, Primary Keys, and Unique Keys)

参考: [https://dev.mysql.com/doc/mysql-partitioning-excerpt/8.0/en/partitioning-limitations-partitioning-keys-unique-keys.html](https://dev.mysql.com/doc/mysql-partitioning-excerpt/8.0/en/partitioning-limitations-partitioning-keys-unique-keys.html)

### !唯一键不能包含空值

比如下面的写法unique_userid不会唯一，可以插入多条deleted_at为null的数据: 
```
CREATE TABLE `pay_flow_qq`
(
    `id`              bigint(20) unsigned NOT NULL AUTO_INCREMENT,
    `user_id`         bigint(20) unsigned DEFAULT NULL COMMENT '用户id',
    `bill_no`         varchar(64) NOT NULL COMMENT '订单号',
    `amount`          bigint(20) DEFAULT NULL COMMENT '发放金额',
    `status`          bigint(20) DEFAULT NULL COMMENT '状态',
    `created_at`      datetime(3) DEFAULT NULL COMMENT '创建时间',
    `updated_at`      datetime(3) DEFAULT NULL COMMENT '更新时间',
    `deleted_at`      datetime(3) DEFAULT NULL COMMENT '删除时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `unique_userid`(`user_id`, `deleted_at`),
    KEY               `idx_userid` ( `user_id`, `deleted_at` ),
    KEY               `idx_billno` ( `bill_no`, `deleted_at` ),
    KEY               `idx_amount` ( `amount`, `deleted_at` ),
    KEY               `idx_status` ( `status`, `deleted_at` ),
    KEY               `idx_pay_flows_deleted_at` (`deleted_at`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4;
```

建议改进方案：
```
    `deleted_at_unix` int (10) DEFAULT 0 COMMENT '删除时间戳',  // 增加这个非空字段
    UNIQUE KEY `unique_userid`(`user_id`, `deleted_at_unix`),  // 用非空字段创建唯一键
```

## 外键约束
### 禁用外键约束:
```
SET FOREIGN_KEY_CHECKS=0
```
### 启动外键约束:
```
SET FOREIGN_KEY_CHECKS=1;
```
### 查看当前FOREIGN_KEY_CHECKS的值可用如下命令:
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
```sql
ALTER TABLE `wenshu_data`.`lawyerinfo` DROP INDEX `idx_lawyerinfo__name`;
```
#### 修改复合索引
```sql
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
```sql
select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from INFORMATION_SCHEMA.TABLES;
```
#### 查看指定数据库大小
```sql
select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from INFORMATION_SCHEMA.TABLES where table_schema='CarData';
```
#### 查看指定数据库的指定表的大小
```sql
select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from INFORMATION_SCHEMA.TABLES where table_schema='CarData' and table_name='driver020294';
```
#### 查看指定数据库指定表的其他大小
```sql
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

```sql
INSERT INTO tablename(field1,field2, field3, ...) VALUES(value1, value2, value3, ...) ON DUPLICATE KEY UPDATE field1=value1,field2=value2, field3=value3, ...;
```

#### 创建utf8mb4数据库

```sql
create database sina default character set utf8mb4 collate utf8mb4_unicode_ci;
```

#### 任务系统表-基于date分区

```sql
CREATE TABLE task.`task_list_default` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '任务id',
  `date` int(11) NOT NULL COMMENT '分区字段',
  `task_type` varchar(64) NOT NULL COMMENT '任务分类',
  `status` tinyint(4) NOT NULL DEFAULT '0' COMMENT '任务状态',
  `channel` varchar(32) NOT NULL COMMENT '任务渠道',
  `parent_ids` varchar(64) DEFAULT '' COMMENT '所依赖的父节点',
  `child_ids` varchar(64) DEFAULT '' COMMENT '被依赖的子节点',
  `begin_time` timestamp NULL DEFAULT NULL COMMENT '任务开始时间',
  `end_time` timestamp NULL DEFAULT NULL COMMENT '任务结束时间',
  `phases` json COMMENT '任务启停重试记录',
  `is_once` bool DEFAULT false COMMENT '是否一次性任务，默认是false',
  `max_retry` tinyint(4) NOT NULL DEFAULT '10' COMMENT '最大重试次数',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `delete_time` timestamp NULL DEFAULT NULL COMMENT '删除时间',
  PRIMARY KEY (`id`, `date`),
  KEY `idx_task_type` (`task_type`) USING BTREE,
  KEY `idx_status` (`status`) USING BTREE,
  KEY `idx_channel` (`channel`) USING BTREE,
  KEY `idx_create_time` (`create_time`) USING BTREE
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='默认渠道任务列表' PARTITION BY RANGE (date)
(PARTITION p202103 VALUES LESS THAN (202104) ENGINE = InnoDB,
 PARTITION p202104 VALUES LESS THAN (202105) ENGINE = InnoDB,
 PARTITION p202105 VALUES LESS THAN (202106) ENGINE = InnoDB,
 PARTITION p202106 VALUES LESS THAN (202107) ENGINE = InnoDB,
 PARTITION p202107 VALUES LESS THAN (202108) ENGINE = InnoDB,
 PARTITION p202108 VALUES LESS THAN (202109) ENGINE = InnoDB,
 PARTITION p202109 VALUES LESS THAN (202110) ENGINE = InnoDB,
 PARTITION p202110 VALUES LESS THAN (202111) ENGINE = InnoDB,
 PARTITION p202111 VALUES LESS THAN (202112) ENGINE = InnoDB,
 PARTITION p202112 VALUES LESS THAN (202201) ENGINE = InnoDB,
 PARTITION p202201 VALUES LESS THAN (202202) ENGINE = InnoDB);
```
```sql
CREATE TRIGGER `task_list_default_date` BEFORE INSERT ON task.`task_list_default` FOR EACH ROW set new.date=date_format(current_date(), '%Y%m');
CREATE TRIGGER `task_list_default_insert` BEFORE INSERT ON task.`task_list_default` FOR EACH ROW set new.create_time=current_date;
CREATE TRIGGER `task_list_default_update` BEFORE UPDATE ON task.`task_list_default` FOR EACH ROW set new.update_time=current_date;
```
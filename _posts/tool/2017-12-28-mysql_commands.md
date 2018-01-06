---
layout: post
title: mysql命令记录
category: 工具
tags: [Mysql命令记录]
keywords: mysql, sql
---

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
#### 添加索引：
```
索引的通常命名： idx_<表名字>__<添加索引字段1>__<添加索引字段2>
CREATE INDEX `idx_cnipsundata__agent` ON `cnipsundata` (`agent`)
```
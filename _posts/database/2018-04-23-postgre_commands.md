---
layout: post
title: postgre命令记录
category: 数据库
tags: [postgre命令记录]
keywords: postgre, sql
---

### 外键
#### 增加外键
```
# 注意user是postgre关键字
ALTER TABLE "user_info" ADD CONSTRAINT user_info_user_id_user_id_foreign FOREIGN KEY (user_id) REFERENCES "user"(id) ON DELETE RESTRICT ON UPDATE RESTRICT;
```

---
title: MySql更新表数据问题分析
date: 2017-03-05 15:58:16
tags: [2017,数据库,mysql]
category: 数据库
---

# 问题
mysql环境执行以下语句：
“You can't specify target table 'mt_datasource' for update in FROM clause”。

不能先select出同一表中的某些值，再update这个表(在同一语句中)
```
-- 更新数据库信息
UPDATE mt_datasource SET is_deleted = 0 where id IN
(
	select id from mt_datasource where id in
		(
		select DISTINCT datasource_id from mt_table
		)
)
```

<!--more-->

# 处理方式
使用中间表规避

## 改写
```
-- 更新数据库信息(逻辑删除)
UPDATE mt_datasource SET is_deleted = 1 where id NOT IN
(
	select a.id from (SELECT * from mt_datasource) a where a.id in
		(
		select DISTINCT datasource_id from mt_table
		)
)
```

---
title: Mysql修改表主键
date: 2017-08-25 14:15:56
tags: Mysql
---

原来有一个字段id,为自增,主键,索引。现在要新增一个字段s_id为自增,主键,索引.同时把原来的主字段改成普通字段,默认值为0.


```
Alter table xxx change s_id s_id int(10) NOT NULL DEFAULT 0;  //去除原来字段的自增属性,不然无法删除这个主键
Alter table xxx drop primary key;  //删除主键
drop index s_id on xxx;  //删除索引,注意这个表原来就只有一个索引

Alter table xxx add column id int(10) NOT NULL DEFAULT 0 FIRST;  //新建一个字段,无法直接新建自增字段,因为不是主键
Alter table xxx add primary key(id); //改为主键,然后才能用自增字段
Alter table xxx change id id int(10) NOT NULL AUTO_INCREMENT;  //改成自增字段
Alter table xxx add UNIQUE INDEX `id` (`id`) USING BTREE ;  //把这个字段改成索引

```

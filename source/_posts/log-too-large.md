---
title: 日志系统膨胀处理
date: 2018-04-25 17:17:12
tags: [log, sql]
---
# 日志系统膨胀处理

## 1.概述
目前的系统有个审计模块，主要是记录大数据系统的各种日志，用来做追溯和审计，并不是数据库本身的日志。
因为日志每次操作都会产生，所以不可避免的带来日志膨胀的问题，为了解决这个问题，必须要定时清理日志。
## 2.处理策略
基本的思路很简单，就是删删删，按照日期，30天或者其他天数前的日志全部都要删除，而且要清除原始log，就是审计模块读取的源log。
## 3.实现策略
最初的想法是在Java工程使用Schedule实现，评估了下觉的略繁琐，而且从架构角度上来看放在Web中搞这个有点别扭，就搜了下在数据库中处理的方法，然后现学现卖。实现分两块，数据库和shell。
* 数据库中定时功能通过事件来实现，然后将各种判断和删除的编写成功函数方便调用。
* shell中将删除和判断的写成脚本，然后通过cron定时调用。

## 4.实现细节
其实思路很简单，实现不太流利，都是现查现试，有些还得做实验，琐碎记录一些点
### 1.MySql自动调用事件
使用navicat（原谅写原生不熟）在“事件”中新建一个自动call的事件
```
call delete_audit;
```
计划标签上Every 1 Day，Start选一个你认为合适的日期。
### 2.MySql自动删除过期数据
依然使用navicat，函数中定义一个delete_audit
```
BEGIN
	delete from bd_hbase_audit where DATE(processDate) <= DATE(DATE_SUB(NOW(), INTERVAL 90 day));
	...
END		
```
### 3. id膨胀问题
问题2的实现是很简单，但是也带来一个问题，数据表的id是自增长的，如果一直删旧的加新的，岂不是会永远膨胀到爆表？目前大概就是这么几个策略。
* 1.这条语句*TRUNCATE TABLE t_books_author;* 表中id是重置了，然后数据也没了，pass。
* 2.仅仅*ALTER TABLE t_books_author auto_increment = 1; *数据是有了，但是新加数据并非从空闲的最小开始，而是还是从已有的最大+1开始，所以，没用。
* 3.懒办法，将id改为bigint，目前的状态，每天增加的日志就那么多，几百年...当然是建立在日志量不会指数级增加的情况下，不算完美解决。
* 4. 建立一个临时的新表，将数据挨个从旧表导入，然后新表id按顺序重新排列。算是笨办法，有点烦，有点慢。
* 5.其实这算方法2的改进版，
```
BEGIN
	set @num := 0;
	update bd_hbase_auth SET id=@num := (@num+1);
	alter table bd_hbase_auth auto_increment=1;
END
```
### 4. 到达某个点再触发reset_id
主要是select到的值怎么到变量里
```
BEGIN
	select id into @id from bd_hbase_auth ORDER BY id desc limit 1;
  if @id>=63 THEN
		call reset_hbase_auth_id;
	end if;
END
```

### 5. 删除过期原始日志
```
invalid_day=90
hdfs_dir="/var/log/hadoop-hdfs/"
hive_dir="/var/log/hive/"
hbase_dir="/var/log/hbase/"

#xargs="xargs rm -f"

find $hdfs_dir -mtime +$invalid_day -type f -name "*hdfs*" | $xargs
find $hive_dir -mtime +$invalid_day -type f -name "*hive*" | $xargs
find $hbase_dir -mtime +$invalid_day -type f -name "*hbase*" | $xargs
```
其中的点一个是mtime的用法，一个是find匹配，用find正则匹配，正则都写好了，找个在线测试也对，但是就是不好使，可能使用有误，转到-name的模糊匹配，先凑合用。

### 6. 定期调用日志
应该直接再cron中添加任务，但是使用脚本部署的时候编辑有点麻烦，正好cron有个cron.daily的文件夹，所以一个cp将步骤5的脚本拷贝过去。

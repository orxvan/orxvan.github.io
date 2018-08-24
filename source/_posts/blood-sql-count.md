---
title:  count引发的血案
date: 2018-04-25 17:28:11
tags: [sql]
---
# count引发的血案

# 1.前情提要
一切在实验平台上都跑的好好的，只是click测试了几下，没有正儿八经的压力测试，这不，部署到一个生产平台，开始发现问题了。log数据飙升，要上亿了，结果就是query页面卡卡卡。不能忍。

# 2.问题
因为有分页，分页要计数，计数用count，然后慢死了。
hdfs_log表的数据量太大了，InnoDB的引擎用count遍历数一次简直要死

# 3.咋办
## 1.explain
```
explain select * from tb_name;
```
返回的rows里有表的数量，速度很快，但是数据是不准确的，关键是*怎么拿到rows*，sql语句怎么写都是不合法：（

## 2.改此表的引擎从InnoDB到MyISAM
在navicat的设计表中直接改，报错，`ERROR 1206 (HY000): The total number of locks exceeds the lock table size`，还是因为数据量太大的问题，要改数据的话，得修改默认配置。
* 查my.cnf的配置文件在哪里`where is mysqld`获取mysqld的路径，
* `/usr/libexec/mysqld --verbose -help| grep -A 1 'Default options'`获得配置文件路径（或者`sudo find / -name 'my.cnf' 2>1`)
* `mysql> show variables like 'innodb_buffer_pool_size';`查看innodb_buffer_pool_size参数
* 在已有的my.cnf中，mysqld下，innodb_buffer_pool_size改为
```
[mysqld]
nnodb_buffer_pool_size=2g
```
* 重启`sudo service mysqld restart`
* 不过最后还是没改，因为觉的这个方法有点不合适，上面的步骤记录的是因为删除大量数据时同样报错，而需要修改的参数。

## 3.从infomation_schema中读
```
select table_rows from information_schema.TABLES where TABLE_NAME='bd_hdfs_log_info';
```
会有4%d误差，数据量很大的时候不可忍受，只有某些情况下可以使用

## 4.自己搞计数器
触发器解决
navicat设计表中，有个触发器，输入个触发器的名字，选择after触发，点选插入
下面的定义中，
```
begin
  update bd_row_tmp set bd_hdfs_log_info_row = bd_hdfs_log_info_row + 1;
end
```
其中，bd_row_tmp是新建的一个表，其中的bd_hdfs_log_info_row是个bigint，记录bd_hdfs_log_info的row数量。

## 5.题外记录
抓原始log的时候，用的java的watch_service，只要一改就能读，按照咱们普通人的理解，肯定写一条log，save下，所以每次读入的log肯定是完整有效的，直接抓就行，实在有个不合适的也无所谓。但是就hdfs的log总是出错，同样的逻辑和代码hive和hbase就没问题，改成不用切分用正则，还是时不时出解析错误的问题，后来我猜测，只能是猜测，写数据没写完，就save，然后watch_service就去读，然后就解析错误了，可以改解析策略加个校验，或者像我一样，加个延时，就能大量减少解析出错的几率。
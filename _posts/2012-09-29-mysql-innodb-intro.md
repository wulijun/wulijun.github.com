---
layout: post
title: "新手必看：一步到位之InnoDB"
description: "MySQL发展到今天，InnoDB引擎已经作为绝对的主力，除了像大数据量分析等比较特殊领域需求外，它适用于众多场景。然而，仍有不少开发者还在“执迷不悟”的使用MyISAM引擎，觉得对InnoDB无法把握好，还是MyISAM简单省事，还能支持快速COUNT(*)。本文是由于最近几天帮忙处理discuz论坛有感而发，希望能对广大开发者有帮助。"
tags: MySQL
---

##快速认识InnoDB

InnoDB是MySQL下使用最广泛的引擎，它是基于MySQL的高可扩展性和高性能存储引擎，从5.5版本开始，它已经成为了默认引擎。
InnODB引擎支持众多特性：

1.	支持ACID，简单地说就是支持事务完整性、一致性； 
2.	支持行锁，以及类似ORACLE的一致性读，多用户并发；
3.	独有的聚集索引主键设计方式，可大幅提升并发读写性能；
4.	支持外键；
5.	支持崩溃数据自修复；

InnoDB有这么多特性，比MyISAM来的优秀多了，还犹豫什么，果断的切换到InnoDB引擎吧 :)

## 修改InnoDB配置选项
可以选择官方版本，或者Percona的分支，如果不知道在哪下载，就google吧。安装完MySQL后，需要适当修改下my.cnf配置文件，针对InnoDB相关的选项做
一些调整，才能较好的运行InnoDB。

相关的选项有：

{% highlight mysql %}
#InnoDB存储数据字典、内部数据结构的缓冲池，16MB 已经足够大了。
innodb_additional_mem_pool_size = 16M

#InnoDB用于缓存数据、索引、锁、插入缓冲、数据字典等
#如果是专用的DB服务器，且以InnoDB引擎为主的场景，通常可设置物理内存的50%
#如果是非专用DB服务器，可以先尝试设置成内存的1/4，如果有问题再调整
#默认值是8M，非常坑X，这也是导致很多人觉得InnoDB不如MyISAM好用的缘故
innodb_buffer_pool_size = 4G

#InnoDB共享表空间初始化大小，默认是 10MB，也非常坑X，改成 1GB，并且自动扩展
innodb_data_file_path = ibdata1:1G:autoextend

#如果不了解本选项，建议设置为1，能较好保护数据可靠性，对性能有一定影响，但可控
innodb_flush_log_at_trx_commit = 1

#InnoDB的log buffer，通常设置为 64MB 就足够了
innodb_log_buffer_size = 64M

#InnoDB redo log大小，通常设置256MB 就足够了
innodb_log_file_size = 256M

#InnoDB redo log文件组，通常设置为 2 就足够了
innodb_log_files_in_group = 2

#启用InnoDB的独立表空间模式，便于管理
innodb_file_per_table = 1

#启用InnoDB的status file，便于管理员查看以及监控等
innodb_status_file = 1

#设置事务隔离级别为 READ-COMMITED，提高事务效率，通常都满足事务一致性要求
transaction_isolation = READ-COMMITTED 
在这里，其他配置选项也需要注意：

#设置最大并发连接数，如果前端程序是PHP，可适当加大，但不可过大
#如果前端程序采用连接池，可适当调小，避免连接数过大
max_connections = 60

#最大连接错误次数，可适当加大，防止频繁连接错误后，前端host被mysql拒绝掉
max_connect_errors = 100000

#设置慢查询阀值，建议设置最小的 1 秒
long_query_time = 1

#设置临时表最大值，这是每次连接都会分配，不宜设置过大 max_heap_table_size 和 tmp_table_size 要设置一样大
max_heap_table_size = 96M
tmp_table_size = 96M

#每个连接都会分配的一些排序、连接等缓冲，一般设置为 2MB 就足够了
sort_buffer_size = 2M
join_buffer_size = 2M
read_buffer_size = 2M
read_rnd_buffer_size = 2M

#建议关闭query cache，有些时候对性能反而是一种损害
query_cache_size = 0

#如果是以InnoDB引擎为主的DB，专用于MyISAM引擎的 key_buffer_size 可以设置较小，8MB 已足够
#如果是以MyISAM引擎为主，可设置较大，但不能超过4G
#在这里，强烈建议不使用MyISAM引擎，默认都是用InnoDB引擎
key_buffer_size = 8M

#设置连接超时阀值，如果前端程序采用短连接，建议缩短这2个值
#如果前端程序采用长连接，可直接注释掉这两个选项，是用默认配置(8小时)
interactive_timeout = 120
wait_timeout = 120
{% endhighlight %}

## 开始使用InnoDB引擎

修改完配置文件，即可启动MySQL。启动完毕后，在MySQL的datadir目录下，若产生以下几个文件，则表示应该可以使用InnoDB引擎了。

	-rw-rw---- 1 mysql mysql 1.0G Sep 21 17:25 ibdata1
	-rw-rw---- 1 mysql mysql 256M Sep 21 17:25 ib_logfile0
	-rw-rw---- 1 mysql mysql 256M Sep 21 10:50 ib_logfile1
	
登录MySQL后，执行命令，确认已启用InnoDB引擎：

	(root:imysql.cn:Thu Oct 15 09:16:22 2009)[mysql]> show engines;
	+------------+---------+----------------------------------------------------------------+--------------+------+------------+
	| Engine     | Support | Comment                                                        | Transactions | XA   | Savepoints |
	+------------+---------+----------------------------------------------------------------+--------------+------+------------+
	| InnoDB     | YES     | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |

接下来创建一个InnoDB表：

{% highlight sql %}
	(root:imysql.cn:Thu Oct 15 09:16:22 2009)[mysql]> 
	CREATE TABLE my_innodb_talbe(
		id INT UNSIGNED NOT NULL AUTO_INCREMENT,
		name VARCHAR(20) NOT NULL DEFAULT '',
		passwd VARCHAR(32) NOT NULL DEFAULT '',
		PRIMARY KEY(id),
		UNIQUE KEY `idx_name`(name)
	) ENGINE = InnoDB;
{% endhighlight %}
	
有几个和MySQL(尤其是InnoDB引擎)数据表设计相关的建议，希望开发者朋友能遵循：

1.	所有InnoDB数据表都创建一个和业务无关的自增数字型作为主键，对保证性能很有帮助；
2.	杜绝使用text/blob，确实需要使用的，尽可能拆分出去成一个独立的表；
3.	时间戳建议使用 TIMESTAMP 类型存储；
4.	IPV4 地址建议用 INT UNSIGNED 类型存储；
5.	性别等非是即非的逻辑，建议采用 TINYINT 存储，而不是 CHAR(1)；
6.	存储较长文本内容时，建议采用JSON/BSON格式存储；

最后，在使用InnoDB过程中如果碰到什么问题，欢迎交流!

原文链接：<http://imysql.com/2012/09/21/mysql-faq-setup-innodb-quickly.html>
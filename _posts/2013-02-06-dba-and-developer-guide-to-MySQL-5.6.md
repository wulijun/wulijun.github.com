---
layout: post
title: "MySQL 5.6之DBA与开发者指南"
description: "介绍MySQL 5.6版本新增的功能和技术改进"
published: true
tags: PHP MySQL
---

## 构建下一代Web应用与服务

简单来说，MySQL 5.6改进了数据库核心的各个功能领域，包括：

*	更好的性能和可伸缩性
	1.	改进InnoDB引擎的事务吞吐量
	2.	改进优化器的查询执行时间和诊断
*	更好的应用可用性，支持在线DDL/Schema修改
*	增强开发者的灵活性，支持通过Memcached API访问InnoDB，实现NoSQL功能
*	改进复制功能，满足高性能，自修复的分布式部署需求
*	改进Performance Schema，更好支持各种新硬件设备
*	改善安全性
*	以及其他一些重要改进

本文作为DBA与开发者的MySQL 5.6指南，介绍了这些重要的新特性，并提供了一些实际用例。

## 更好的性能和可伸缩性: InnoDB存储引擎的改进

从运维的角度来看，MySQL 5.6在多处理器和高CPU并发线程的系统上，性能和可伸缩性有更好的持续线性增长能力。
原因是Oracle的InnoDB存储引擎移除了遗留的线程争用和mutex锁，提升了效率和并发度。这些改进使得MySQL可以充分
利用x86-based COTS(commodity-off-the-shelf)硬件的高级多线程处理能力。

内部的读写和只读负载测试数据表明，MySQL 5.6的线性扩展能力明显超过5.5版本。下图显示了MySQL 5.6在并发CPU线程
增加到60时每秒读写事务数TPS的线性增长状况。

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_scalability_rw.png">

只读TPS的持续线性增长状况见下图

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_scalability_ro.png">

## 更好的事务吞吐量

MySQL 5.6改进了高并发、事务型和读密集负载的性能和可伸缩性。这些用例中，性能改进主要体现于在并发用户不断增长的
情况下，应用服务的表现和可伸缩性。InnoDB重构了架构，减少mutex争用和瓶颈，提供对底层数据更加一致的访问路径，这些
改进包括：

*	拆分核心mutex，去除单点争用
*	flushing操作使用新线程
*	新的多线程purge
*	新的自适应哈希算法
*	更少的buffer pool争用
*	以更加频繁且可预测的间隔收集与持久化优化器统计数据，从而生成更好更加一致的查询执行

SysBench read/write性能测试展现了这些改进的结果：

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_connections_rw.png">

在Linux平台上，MySQL 5.6的TPS吞吐量比5.5版本提高了150%，在Windows 2008平台上，大约提高了47%。

## 更好的只读负载的吞吐量

InnoDB针对只读型事务做了新的优化，去掉了事务的开销，对基于web的查询和报表类应用，可以极大地提升性能。这些优化在
autocommit=1时默认开启，另外开发者也可以通过START_TRANSACTION_READ_ONLY语句开启：

> SET autocommit = 0;
> START_TRANSACTION_READ_ONLY;
> SELECT c FROM T1 WHERE id=N;
> COMMIT;

优化后的结果如下图：

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_connections_ro.png">

在Linux平台上，MySQL 5.6的只读TPS吞吐量比5.5版本提高了230%，在Windows 2008平台上，大约提高了65%。

上述性能测试的运行平台配置如下：

*	Oracle Linux 6
*	Intel® Xeon(R) E7540 x86_64
*	MySQL leveraging:
	1.	60 of 96 available CPU threads
	2.	2 GHz, 512GB RAM

测试套件SysBench是用于具体应用用例性能测试的免费工具，下载地址<http://dev.mysql.com/downloads/benchmarks.html>

如果对MySQL 5.6性能和各特性的性能测试感兴趣，可以参考[Mikael Ronstrom的博客](http://mikaelronstrom.blogspot.com/)
和[Dimitri Kravtchuk's blog](http://dimitrik.free.fr/blog/)，他们分享了测试结果，并提供了测试时使用的测试用例和配置。

## 更好支持固态硬盘(SSD)

普通硬盘经常成为各种系统的瓶颈，原因是它的物理特性的限制，使得在高并发下很难有好的可伸缩性。因此，许多需要支持高并发的
web应用，它们的MySQL会部署在SSD上，从而获得既可靠，访问速度又和内存相似的服务。MySQL 5.6包含了几个重要的改进，以支持SSD
这类设备：

*	支持4k和8k的页大小，以适应SSD的标准存储算法
*	便携式的.ibd(InnoDB数据)文件，允许InnoDB“热”表可以从默认的数据目录下移到SSD或者网络存储设备
*	InnoDB undo日志可使用独立表空间，支持把undo日志从系统表空间移到一个或多个独立的表空间。如undo日志的读密集I/O模式，就可以把undo日志移到SSD，而系统表空间仍然放在普通硬盘。

[学习更多SSD优化技术](http://dev.mysql.com/doc/refman/5.6/en/innodb-performance.html)

## 更好的查询执行时间和诊断：优化器的改进

MySQL 5.6的优化器进行了重构，提升了效率和性能，主要改进有：

### 子查询优化
通过半连接和物化技术，MySQL优化器提高了子查询的性能，简化开发者编写查询的复杂度。特别是From子句中的子查询，只在需要子查询内容时才执行物化以提升性能；同时优化器会在必要的时候，给派生表添加索引以加快记录读取速度。使用DBT-3 benchmark Query #13语句测试，表明性能比之前版本有了很大的提高。

{% highlight sql %}
select c_name, c_custkey, o_orderkey, o_orderdate, o_totalprice, sum(l_quantity)
from customer, orders, lineitem
where o_orderkey in (
                select l_orderkey
                from lineitem
                group by l_orderkey
                having sum(l_quantity) > 313
  )
  and c_custkey = o_custkey
  and o_orderkey = l_orderkey
group by c_name, c_custkey, o_orderkey, o_orderdate, o_totalprice
order by o_totalprice desc, o_orderdate
LIMIT 100;
{% endhighlight %}

更多详情参考 [From Months to Seconds with Subquery Materialization](http://oysteing.blogspot.com/2012/07/from-months-to-seconds-with-subquery.html)

### 针对较小Limit的文件排序优化
对于有ORDER BY和较小LIMIT值的查询，优化器现在通过单遍表扫描就能生成有序结果集。这种查询在Web应用中比较常见，用于显示一个大结果集中的少数记录，如下示例：
> SELECT col1, ... FROM t1 ... ORDER BY name LIMIT 10;
内部测试显示，该优化最大可以提升4倍性能，大大优化了用户体验和响应时间。更多详情参考[博客](http://didrikdidrik.blogspot.com/2011/04/optimizing-mysql-filesort-with-small.html)

### 索引条件下推 (ICP)
优化器现在默认把where条件下推到存储引擎进行求值、表扫描和返回有序结果集给MySQL server。

{% highlight sql %}
CREATE TABLE person (
      personid INTEGER PRIMARY KEY,
      firstname CHAR(20),
      lastname CHAR(20),
      postalcode INTEGER,
      age INTEGER,
      address CHAR(50),
      KEY k1 (postalcode,age)‏
   ) ENGINE=InnoDB;

SELECT lastname, firstname FROM person
   WHERE postalcode BETWEEN 5000 AND 5500 AND age BETWEEN 21 AND 22;
{% endhighlight %}

内部测试显示该类表的这种查询，ICP优化最大可以提升15倍性能。

### 批量键访问(BKA)和多范围读(MRR)
现在优化器把所有主键批量提供给存储引擎，使得它可以更有效的访问、排序和返回数据，减少查询执行时间。

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_bka.png">

对于DBT-3 Query 13和其他磁盘密集型查询语句的测试显示，BKA和MRR最大可以提高280倍的性能。更多详情参考[Batched Key Access Speeds Up Disk-Bound Join Queries](http://oysteing.blogspot.com/2011/10/bacthed-key-access-speeds-up-disk-bound.html)。

### 优化器诊断的改进 - MySQL 5.6优化器在诊断和调试方面的改进

*	EXPLAIN for INSERT, UPDATE, and DELETE operations,
*	EXPLAIN plan output in JSON format with more precise optimizer metrics and better readability
*	Optimizer Traces for tracking the optimizer decision-making process.

[Learn about all of MySQL 5.6 Optimizer improvements and features, along with all technical documentation](http://dev.mysql.com/refman/5.6/en/mysql-nutshell.html)

如果想了解实现细节、如何使用与使用例子，请阅读[MySQL优化器工程团队的博客](http://mysqloptimizerteam.blogspot.co.uk/)

## 应用可用性的改进：在线DDL/Schema修改
如今基于Web的应用都需要快速演进，以适应业务需求。并且对服务等级协议(SLA)也是以分钟、天或周来衡量。因此当应用需要快速支持新产品线或者对现有产品进行升级时，后端数据库Schema也需要能够平滑升级。MySQL 5.6为ALTER TABLE增加了如下DDL语法，提升在线Schema的灵活度和敏捷度。

*	CREATE INDEX
*	DROP INDEX
*	Change AUTO_INCREMENT value for a column
*	ADD/DROP FOREIGN KEY
*	Rename COLUMN
*	Change ROW FORMAT, KEY_BLOCK_SIZE for a table
*	Change COLUMN NULL, NOT_NULL
*	Add, drop, reorder COLUMN

DBA和开发者可以在线添加/删除索引和执行标准的InnoDB表修改而无需停服务，可以极大地方便开发者灵活修改Schema以适应新的业务需求。

更多MySQL 5.6 InnoDB online DDL改进和特性，请参考[文档](http://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl.html)

## 敏捷开发支持：InnoDB支持NoSQL访问
当前很多web、云、社交和移动应用都需要这样的服务：既能够对数据执行快速的Key/Value操作，又能保证这些数据的ACID特性，能执行复杂的查询。通过InnoDB的NoSQL API，开发者就可以同时拥有传统RDBMS的特性和高性能的KV查询能力。

MySQL 5.6提供常见的Memcached API和InnoDB进行简单的KV操作。它在mysqld中包含了Memcached后台插件，通过Memcached协议直接和InnoDB原生API交互，绕过消耗很大的查询分析阶段，进行InnoDB数据的查询和执行兼容事务的数据更新。该API把Memcached功能集成在持久化、崩溃安全、事务型的数据库中，并兼容原有的标准Memcached库和客户端。实现如下图所示：

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_nosql.png">

这么做和普通SQL的性能差距有多大？内部性能测试显示，针对某些场景，SET/INSERT操作可以提高9倍的吞吐量：

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_nosql_benchmarks.png">

这对开发者和DBA而言，不仅可以提高性能和灵活性，还可以减少复杂性，原来分离的cache和数据库层，现在可以放在一个数据层中，还能解决数据一致性的问题。

要了解更多详情，请参考[InnoDB team blog](https://blogs.oracle.com/mysqlinnodb/entry/new_enhancements_for_innodb_memcached)

[Learn more about the details and how to get started with the new Memcached API to InnoDB](http://dev.mysql.com/doc/refman/5.6/en/innodb-memcached.html)

## 敏捷开发支持：扩展InnoDB适用场景
MySQL 5.6的优化和新特性，扩展了MySQL的适用场景，开发者可以仅使用InnoDB一种存储引擎，就能完成多种任务，从而简化应用的开发。

### 新的全文搜索 (FTS)
作为MyISAM FTS的替代者，InnoDB支持对文本内容创建FULLTEXT索引，加快词语和短语的搜索。InnoDB全文搜索支持自然语言/布尔模式、近似搜索和相关性排序。下面是一个示例：

{% highlight sql %}
CREATE TABLE quotes
(id int unsigned auto_increment primary key
, author varchar(64)
, quote varchar(4000)
, source varchar(64)
, fulltext(quote)
) engine=innodb;

SELECT author AS "Apple" FROM quotes
    WHERE match(quote) against (‘apple' in natural language mode);
{% endhighlight %}

### 新的可迁移表空间
在file-per-table模式下创建的InnoDB .ibd文件，现在可以在不同的物理存储设备和数据库服务器间迁移，开发者可以在创建表的时候，可以为.ibd文件指定一个不在MySQL数据目录下的存储位置。这个特性使得开发者可以把“热”表迁移到外部网络存储设备(如SSD和HDD)中，降低服务器负载。并且可以简单的导出/导入InnoDB表，从而快速、无缝的伸缩应用，如下例所示：

导出:

{% highlight sql %}
CREATE TABLE t(c1 INT) engine=InnoDB;
FLUSH TABLE t FOR EXPORT; -- quiesce the table and create the meta data file
$innodb_data_home_dir/test/t.cfg
UNLOCK TABLES;
{% endhighlight %}

导入:

{% highlight sql %}
CREATE TABLE t(c1 INT) engine=InnoDB; -- if it doesn't already exist
ALTER TABLE t DISCARD TABLESPACE;
-- The user must stop all updates on the tables, prior to the IMPORT
ALTER TABLE t IMPORT TABLESPACE;
{% endhighlight %}

更多InnoDB的改进，请参考[文档](http://dev.mysql.com/doc/refman/5.6/en/mysql-nutshell.html)

## 优化复制和高可用性
复制是MySQL能够可伸缩和高可用性的关键特性，MySQL 5.6新增自修复式复制拓扑和高性能的主从服务，使得开发者能够构建下一代的新应用。

### 新增全局事务标识 (GTID)
GTID用于跟踪主从复制拓扑中的事务完整性，为自修复式恢复提供了基础，而且也方便DBA和开发者在主库失败时找到复制延时最小的从库。GTID直接保存在Binlog中，再也不需要像以前版本那样，需要借助复杂的第三方插件才能完成类似的任务。

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_gtid.png" title="GTID positioning in the Binlog" alt="GTID positioning in the Binlog">

### 新增MySQL复制工具集
MySQL 5.6版本提供一组Python编写的用于管理和监控主从复制的工具，利用GTID，实现主库失败时自动fail-over功能与维护时主库切换功能，不再依赖第三方的高可用性方案，并且不需要OP人工干预，减少服务宕机时间。

下载白皮书: [MySQL 复制: 高可用性 - 构建自修复式复制拓扑](http://www.mysql.com/why-mysql/white-papers/mysql-replication-high-availability)

### 新增多线程从库
根据Schema划分工作线程，允许并行更新。对于那些使用不同数据库分割应用的系统，效率可以获得很大的提升，如多租户单实例系统(multi-tenant systems)。

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_mts.png">

SysBench benchmarks在10个Schema上使用多个工作线程的测试结果表明，性能可以最大提升5倍左右。

### 新增Binary Log Group Commit (BGC)
MySQL 5.6主库在复制时按组写入Binlog，而不是逐个提交，极大地提升了主库性能。BGC同时也减少了锁等待，对性能也有提升，测试结果如下图：

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_blgc.png">

MySQL 5.6的吞吐量相比5.5版本有180%左右的提升。BGC让开发者不用再像以前那样，在主库性能和MySQL复制提供的可伸缩、高可用性之间做艰难的抉择了。

*	新优化的基于行的复制 - MySQL 5.6新增选项binlog-row-image=minimal，只复制DML操作中修改的数据元素，提升主从复制吞吐量，减少binlog大小、网络资源和服务器内存占用。
*	新增Crash-Safe从库 - MySQL 5.6在表中保存Binlog位置信息，在复制失败时从库可以自动回滚到最后提交的状态，并重启复制进程而无需人工干预。这不仅减少操作负担，也防止从库从错误的数据文件进行恢复导致的数据丢失，而且当主库崩溃，导致Binlog出错时，服务器也能自动回退到最后的正确状态。
*	新增复制校验和 - MySQL 5.6从库会检测数据是否损坏并返回错误，保证从库上的数据完整性，防止从库被错误的数据破坏。
*	新增延时复制 - MySQL 5.6允许开发者设置复制延时时间，防止主库的错误操作扩散到相应的从库。在到达这个延时之前，如果主库出现问题， With configurable master to slave time delays, in the event of failure or mishap, slaves can be promoted to the new master in order to restore the database to its previous state. It also becomes possible to inspect the state of a database before an error or outage without the need to reload a back up.

更多MySQL 5.6复制和高可用性改进和特性，请参考[文档](http://dev.mysql.com/doc/refman/5.6/en/mysql-nutshell.html)和[Mat Keep's Developer Zone article](http://dev.mysql.com/tech-resources/articles/mysql-5.6-replication.html)

最后，更多MySQL复制操作指导，可以参考下面这些资源：

*	[White Paper: Introduction to MySQL Replication](http://www.mysql.com/why-mysql/white-papers/mysql-replication-introduction)
*	[Using MySQL Replication for Highly Available Applications](http://www.mysql.com/why-mysql/white-papers/mysql-replication-high-availability)
*	[MySQL Replication Hands On Tutorial: Configuration, Provisioning and Management](http://www.mysql.com/why-mysql/white-papers/mysql-replication-tutorial.)

## Improved Performance Schema
MySQL Performance Schema在MySQL 5.5引入，用于查看关键性能指标。MySQL 5.6增强了Performance Schema的功能，提供了DBA和开发者常见问题的答案。包括：

*	Statements/Stages
	*	哪些是资源密集型查询？时间花在哪里？
*	Table/Index I/O, Table Locks
	*	哪些表和索引负载最高或者争用最多？
*	Users/Hosts/Accounts
	*	哪些用户、主机与帐号最费资源？
*	Network I/O
	*	网络负载如何？会话空闲时间如何？
*	Summaries
	*	Aggregated statistics grouped by statement, thread, user, host, account or object.

MySQL 5.6 Performance Schema在my.cnf默认启用，不过各项配置已经优化，占用资源不多(少于5%，不同场景有区别)，因此可以在线上产品使用。In addition, new atomic levels of instrumentation enable the capture of granular levels of resource consumption by users, hosts, accounts, applications, etc. for billing and chargeback purposes in cloud computing environments.

MySQL Engineering has several champions behind the 5.6 Performance Schema, and many have published excellent blogs that you can reference for technical and practical details. To get started see [Mark Leith's blog](http://www.markleith.co.uk/2012/07/) and [Marc Alff's blog](http://marcalff.blogspot.com/).

[The MySQL docs are also an excellent resource for all that is available and that can be done with the 5.6 Performance Schema](http://dev.mysql.com/doc/refman/5.6/en/performance-schema.html).

## 安全性改进
MySQL 5.6 introduces a major overhaul to how passwords are internally handled and encrypted. The new options and features include:

*	New alternative to password in master.info - MySQL 5.6 extends the replication START SLAVE command to enable DBAs to specify master user and password as part of the replication slave options and to authenticate the account used to connect to the master through an external authentication plugin (user defined or those provided under MySQL Enterprise Edition). With these options the user and password no longer need to be exposed in plain text in the master.info file.
*	New encryption for passwords in general query log, slow query log, and binary log - Passwords in statements written to these logs are no longer recorded in plain text.
*	New password hashing with appropriate strength - Default password hashing for internal MySQL server authentication via PASSWORD() is now done using the SHA-256 password hashing algorithm using a random salt value.
*	New options for passwords on the command line - MySQL 5.6 introduces a new "scrambled" option/config file (.mylogin.cnf) that can be used to securely store user passwords that are used for command line operations.
*	New change password at next login - DBAs and developers can now control when account passwords must be changed via a new password_expired flag in the mysql.user table.
*	New policy-based Password validations - Passwords can now be validated for appropriate strength, length, mixed case, special chars, and other user defined policies based on LOW, MEDIUM and STRONG designation settings. See /doc/refman/5.6/en/validate-password-plugin.html for details and available configuration options.

[Learn about these and all of MySQL 5.6 Security improvements and features, along with all technical documentation](http://dev.mysql.com/doc/refman/5.6/en/mysql-nutshell.html).

## 其他改进
*	新的默认配置优化 - MySQL 5.6针对当前系统架构，修改了服务器配置项的默认值，提高默认配置下的服务器性能。这些新值适合常见场景，省去了手动更改的麻烦。

	修改的配置项、自动设置的配置项和启动时可以设置的配置项清单，请[查看服务器默认配置](http://dev.mysql.com/doc/refman/5.6/en/server-default-changes.html)。
*	改进TIME/TIMESTAMP/DATETIME数据类型：
	*	TIME/TIMESTAMP/DATETIME - 现在允许微秒级别的时间/日期的比较和数据选取
	*	TIMESTAMP/DATETIME - 允许赋值为当前时间戳/自动更新值，或两者作为TIMESTAMP和DATETIME列的默认值。
	*	TIMESTAMP - 现在默认值是null。TIMESTAMP列如果没有显式指定，不再取NOW()作为默认值，如果指定为非null且没有显式指定默认值，那么当作没有默认值处理。
*	Better Condition Handling - GET DIAGNOSTICS
*	MySQL 5.6 enables developers to easily check for error conditions and code for exceptions by introducing the new MySQL Diagnostics Area and corresponding GET DIAGNOSTICS interface command. The Diagnostic Area can be populated via multiple options and provides 2 kinds of information:
	*	Statement - which provides affected row count and number of conditions that occurred
	*	Condition - which provides error codes and messages for all conditions that were returned by a previous operation

The addressable items for each are:

<img src="http://dev.mysql.com/tech-resources/articles/mysql-5.6/mysql_56_diagnostics.png">

The new GET DIAGNOSTICS command provides a standard interface into the Diagnostics Area and can be used via the CLI or from within application code to easily retrieve and handle the results of the most recent statement execution:

> mysql> DROP TABLE test.no_such_table;
> ERROR 1051 (42S02): Unknown table 'test.no_such_table'
> mysql> GET DIAGNOSTICS CONDITION 1
> -> @p1 = RETURNED_SQLSTATE, @p2 = MESSAGE_TEXT;
> mysql> SELECT @p1, @p2;
> +-------+------------------------------------+
> | @p1   | @p2                                |
> +-------+------------------------------------+
> | 42S02 | Unknown table 'test.no_such_table' |
> +-------+------------------------------------+

Options for leveraging the MySQL Diagnotics Area are detailed in the [MySQL Diagnostics documentation](http://dev.mysql.com/doc/refman/5.6/en/diagnostics-area.html). GET DIAGNOTICS is well documented in the [Get Diagnostics documentation](http://dev.mysql.com/doc/refman/5.6/en/get-diagnostics.html).

*	Improved IPv6 Support
	*	MySQL 5.6 improves INET_ATON() to convert and store string-based IPv6 addresses as binary data for minimal space consumption.
	*	MySQL 5.6 changes the default value for the bind-address option from "0.0.0.0" to "0::0" so the MySQL server accepts connections for all IPv4 and IPv6 addresses. You can learn more [in the MySQL Documentation](http://dev.mysql.com/doc/refman/5.6/en/server-options.html#option_mysqld_bind-address)
*	分区功能改进
	*	改进包含大量分区的表的性能 - MySQL 5.6现在在包含大量分区的表上的性能和可伸缩性都有很大改进，特别是插入操作，可以高效的对数百个分区操作。
	*	Import/export tables to/from partitioned tables - MySQL 5.6支持用户通过ALTER TABLE ... EXCHANGE PARTITION语句修改表分区或子分区，分区或子分区的行可以移到未分区的表，反向操作也可以。更多细节参考[MySQL Documentation](http://dev.mysql.com/doc/refman/5.6/en/partitioning-management-exchange.html).
	*	显式选择分区 - MySQL 5.6可以显示指定分区或子分区，检索这些分区中的满足WHERE条件的数据行。和自动分区裁剪类似，检索的分区由语句提供者指定。该操作支持查询语句和其他一些DML语句(SELECT, DELETE, INSERT, REPLACE, UPDATE, LOAD DATA, LOAD XML)。更多内容请参考[MySQL Documentation](http://dev.mysql.com/doc/refman/5.6/en/partitioning-selection.html)
*	Improved GIS: Precise spatial operations - MySQL 5.6 provides geometric operations via precise object shapes that conform to the OpenGIS standard for testing the relationship between two geometric values. Learn more by referencing the [MySQL Documentation](http://dev.mysql.com/doc/refman/5.6/en/functions-for-testing-spatial-relations-between-geometric-objects.html#functions-that-test-spatial-relationships-between-geometries)

## 总结
MySQL 5.5曾被称为已经发布的最好的MySQL版本，现在MySQL 5.6则在此之上对性能、可伸缩性、事务吞吐量与可用性等方面进行了改进，以满足web、云和嵌入式使用场景的需求。MySQL 5.6现在已经正式发布，你可以在 is now Generally Available and you can download the fully-functioning, production-ready product from the [MySQL Developer Zone](http://dev.mysql.com/downloads/mysql/)下载功能完整、产品级的MySQL。

如前所述，本文只是介绍了MySQL 5.6的主要特性，完整的变化，请查阅[MySQL Documentation](http://dev.mysql.com/doc/refman/5.6/en/mysql-nutshell.html)。

原文链接：<http://dev.mysql.com/tech-resources/articles/mysql-5.6.html>


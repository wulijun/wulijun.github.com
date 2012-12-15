---
layout: post
title: "MongoDB PHP Driver的连接处理"
description: "分析MongoDB的PHP Driver的连接处理策略，详细分析1.3版本相对于1.2版本驱动的策略变化内容。"
published: false
tags: PHP MongoDB
---

##MongoDB PHP Driver的连接处理

1.3版本的PHP MongoDB driver重写了连接处理库，和以前版本相比，在持久连接和连接池方面，都有了重大的变化。

### 1.2版本的连接管理

1.2版本的驱动引入了连接池，在执行任何查询时，都会从连接池中请求一个连接，完成之后再归还给连接池。这里的完成是指持有该连接的变量离开了它的作用域，下面是一个示例。

最简单的版本:

{% highlight php %}
<?php
$m = new MongoClient();    // ← 从连接池请求连接
$c = $m->demo->test;
$c->insert( array( 'test' => 'yes' ) );
?>
{% endhighlight %}
← $m离开作用域，连接归还给连接池

在函数中:

{% highlight php %}
<?php
function doQuery()
{
        $m = new MongoClient();    // ← 从连接池请求连接
        $c = $m->demo->test;
        $c->insert( array( 'test' => 'yes' ) );
} // ← $m离开作用域，连接归还给连接池
?>
{% endhighlight %}

在某些情况下，系统可能会产生大量的连接，比如在ORMs/ODMs的某个复杂结构中引用连接对象，如下例子：

{% highlight php %}
<?php
for ( $i = 0; $i < 5; $i++ )
{
        $conns[] = new MongoClient();
}// ← 现在有5个连接
?>
{% endhighlight %}

### 1.3版本的连接管理

在1.3版本中，连接管理做了很大改动。每个worker进程(线程、PHP-FPM或Apache worker)中，驱动把连接管理和Mongo*对象分离，降低驱动的复杂度。下面以单个节点的MongoDB实例来说明驱动如何处理连接。

当一个worker进程启动，MongoDB驱动会为之初始化连接管理器管理连接，并且默认没有连接。

在第一个请求调用new MongoClient();时，驱动创建一个新连接，并且以一个哈希值标识这个连接。这个哈希值包括以下参数：主机名、端口，进程ID和可选的replica set名，如果是密码验证的连接，则还包括数据库名、用户名和密码的哈希值（对于密码验证的连接，我们后面再详细讨论）。调用MongoClient::getConnections()方法，可以查看连接对应的哈希值：

{% highlight php %}
<?php
$m = new MongoClient( 'mongodb://whisky:27017/' );
var_dump( $m->getConnections()[0]['hash'] );
?>
{% endhighlight %}

输出:

> string(22) "whisky:27017;-;X;22835"

输出中的"-"表示该连接不属于某个replica set，"X"是没有用户名、数据库和密码时的占位符，22835是当前进程的进程ID。

然后该连接会在连接管理器中注册：
<img src="http://derickrethans.nl/images/content/manager-1con.png">

在需要连接的任何时候，包括插入、删除、更新、查找或执行命令，驱动都会向管理器请求一个合适的连接来执行。请求连接时会用到new MongoClient()的参数和当前进程的ID。每个worker进程/线程，连接管理器都会有一个连接列表，而每个PHP worker同一时刻，只会运行一个请求，因此和每个MongoDB之间只需要一个连接，不断重用，直到PHP worker终止或显式调用MongoClient::close()关闭连接。

### Replica sets

在存在复制集的环境中，情形有点不一样。new MongoClient()的连接字符串中，需要指定多个hosts，并标示当前正在实用复制集:

> $m = new MongoClient("mongodb://whisky:13000,whisky:13001/?replicaSet=seta");

其中的replicaSet参数不能省略，否则驱动会认为你是准备连接三个不同的mongos进程。

在实例化时，驱动会检查复制集的拓扑结构。下面例子的输出，显示在调用new MongoClient()之后，复制集中所有可见的数据节点都会在管理器中注册一个连接:

{% highlight php %}
<?php
$m = new MongoClient( 'mongodb://whisky:13001/?replicaSet=seta' );
foreach ( $m->getConnections() as $c )
{
	echo $c['hash'], "\n";
}
?>
{% endhighlight %}
输出：

> whisky:13001;seta;X;32315
> whisky:13000;seta;X;32315

虽然连接字符串中没有whisky:13000节点，但是管理器中已经注册了两个连接：

<img src="http://derickrethans.nl/images/content/manager-replcon.png">

管理器不仅包含连接的哈希值和TCP/IP socket，还保存哪个节点是主节点，以及每个节点的“距离"。下面的脚本显示了这些额外的信息；

{% highlight php %}
<?php
$m = new MongoClient( 'mongodb://whisky:13001/?replicaSet=seta' );
foreach ( $m->getConnections() as $c )
{
	echo $c['hash'], ":\n",
		" - {$c['connection']['connection_type_desc']}, ",
		"{$c['connection']['ping_ms']} ms\n";
}
?>
{% endhighlight %}

输出：

> whisky:13001;seta;X;5776:
>  - SECONDARY, 1 ms
> whisky:13000;seta;X;5776:
> - PRIMARY, 0 ms

驱动把操作分为两种类型：写操作，包括插入、更新、删除和命令；读操作，包括find和findOne。默认情况下，如果没有设置读偏好参数，管理器会一直返回主节点的连接。读偏好参数可以通过setSlaveOkay()设置，也可以在连接字符串中设置：

> $m = new MongoClient("mongodb://whisky:13000,whisky:13001/?replicaSet=seta&readPreference=secondaryPreferred");

加上这些参数后，连接字符串变得特别长，因此PHP驱动允许将选项放在数组中，作为第二个参数传入：

{% highlight php %}
$options = array(
        'replicaSet' => 'seta',
        'readPreference' => 'secondaryPreferred',
);
$m = new MongoClient("mongodb://whisky:13000,whisky:13001/", $options);
{% endhighlight %}

对于每个操作，驱动向管理器请求获取一个合适的连接。对于写操作，会一直返回主节点的连接；对于读操作，如果辅助节点可用且“距离”不远的话，则会返回该辅助节点的连接。

### 验证的连接

如果MongoDB启用验证功能，那么连接的哈希值会包含验证相关的哈希值。这样不同脚本，使用不同的用户名、密码连接同一个MongoDB上的不同的数据库时，能够相互区分，而不会误用连接。下面示例使用admin用户名连接admin数据库，然后观察hash值的变化：

{% highlight php %}
<?php
$m = new MongoClient( 'mongodb://admin:admin@whisky:27017/admin' );
var_dump( $m->getConnections()[0]['hash'] );
?>
{% endhighlight %}
输出:

> string(64) "whisky:27017;-;admin/admin/bda5cc70cd5c23f7ffa1fda978ecbd30;8697"

以前示例中的"X"部分已经替换为一个包含数据库名admin、用户名admin和哈希值bda5cc70cd5c23f7ffa1fda978ecbd30，该哈希值是根据用户名、数据库名和密码哈希值计算得来。

为了验证能够正确工作，需要在连接字符串中包含数据库名，否则会默认为admin。

在建立连接后要使用数据库，需要先选择该数据库，如：

> $collection = $m->demoDb->collection;
> $collection->findOne();

如果选择的数据库是连接字符串中指定的数据库，或者连接字符串中的数据库是admin，那么一切都会正常运行。否则，驱动会创建一个新的连接，从而防止验证被绕过，如下所示：

{% highlight php %}
<?php
$m = new MongoClient( 'mongodb://user:user@whisky:27017/test' );

$db = $m->test2;
$collection = $db->collection;
var_dump( $collection->findOne() );
?>
{% endhighlight %}
输出:

> Fatal error: Uncaught exception 'MongoCursorException' with message
> 'whisky:27017: unauthorized db:test2 ns:test2.collection lock type:0 client:127.0.0.1'
> in …/mongo-connect-5.php.txt:6

因为我们的连接并没有执行test2数据库的授权验证，因而失败。如果我们执行验证，就会正常运行：

{% highlight php %}
<?php
$m = new MongoClient( 'mongodb://user:user@whisky:27017/test' );

$db = $m->test2;
$db->authenticate('user2', 'user2' );
$collection = $db->collection;
$collection->findOne();

foreach ( $m->getConnections() as $c )
{
	echo $c['hash'], "\n";
}
?>
{% endhighlight %}
输出:

> whisky:27017;-;test/user/602b672e2fdcda7b58a042aeeb034376;26983
> whisky:27017;-;test2/user2/984b6b4fd6c33f49b73f026f8b47c0de;26983

现在管理器中有两个已验证的连接：

<img src="http://derickrethans.nl/images/content/manager-auth.png">

顺便提一句，如果你打开了E_DEPRECATED级别的错误提示，则会看到:

> Deprecated: Function MongoDB::authenticate() is deprecated in …/mongo-connect-6.php.txt on line 5

驱动建议通过创建两个MongoClient对象完成该类任务：

{% highlight php %}
<?php
$mTest1 = new MongoClient( 'mongodb://user:user@whisky:27017/test', array( 'connect' => false ) );
$mTest2 = new MongoClient( 'mongodb://user2:user2@whisky:27017/test2', array( 'connect' => false ) );

$mTest1->test->test->findOne();
$mTest2->test2->test->findOne();

foreach ( $mTest2->getConnections() as $c )
{
	echo $c['hash'], "\n";
}
?>
{% endhighlight %}

单个MongoDB服务器能支持的并发连接相当有限，如果使用PHP-FPM的话，每个worker进程有自己独立的连接池，那么很容易达到连接数的上限。因此，在生产环境中，不管有没有使用复制集，都要部署mongos，然后PHP-FPM连接mongos，这样可以减少mongod的连接数，并且PHP-FPM和mongos之间可以使用短连接(即每个请求结束时都显式调用close函数关闭MongoDB连接)。

原文链接：<http://derickrethans.nl/mongodb-connection-handling.html>


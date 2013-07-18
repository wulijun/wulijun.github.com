---
layout: post
title: "PHP5.5新特性介绍"
description: "最近PHP5.5已经发布，引入了一些新特性。本文将介绍这些特性，并讨论它们可以给开发者带来哪些好处"
tags: PHP
---

最近PHP5.5已经发布，引入了一些新特性。本文将介绍这些特性，并讨论它们可以给开发者带来哪些好处。

## 生成器(Generators)

生成器是其中最令人期待的一个新特性，它使得开发者无需实现迭代器接口，就能实现遍历功能。编写一个实现迭代器接口的类，需要拷贝很多重复
的代码，现在使用生成器，就可以减少代码量和复杂度。

生成器通过新增的关键字yield实现，外形和普通函数类似，但是和函数只返回单个值不同的是，生成器可以生成任意个值。下面通过一个例子展示其
强大功能。考虑PHP中的range()函数，它返回介于$start和$end之间的数值数组，如以下用法：
{% highlight php %}
<?php
foreach (range(0, 1000000) as $number) {
    echo $number;
}
{% endhighlight %}
这个例子中range()函数返回的数组将会占用大量的内存(大约100多Mb)，撇开这个简单的例子，现实程序中，也经常需要创建巨大的数组，它们耗费
大量的时间和内存。引入生成器之后，我们就无需编写迭代器类，也能解决这个问题。生成器不会创建一个大数组，而是每次迭代的时候返回一个值。
上面这个例子可以改成使用生成器的版本：
{% highlight php %}
<?php
// define a simple range generator
function generateRange($start, $end, $step = 1) {
    for ($i = $start; $i < $end; $i += $step) {
        // yield one result at a time
        yield $i;
    }
}

foreach (generateRange(0, 1000000) as $number) {
    echo $number;
}
{% endhighlight %}
这个代码的运行结果和第一个例子一样，但是运行过程中不会创建一个大数组保存所有的值，因而只需不到1K的内存，比原来的代码大大节省了内存
的消耗。

## 密码哈希

新的密码哈希API是PHP5.5中非常重要和实用的特性。以前，开发者只能依赖其他的crypt()函数，而这些函数的文档又不是很齐全，导致很多误用。
现在新增的API简单明了，方便开发者实现安全的密码哈希功能。

新API包含password_hash()和password_verify()等函数，调用password_hash($password, PASSWORD_DEFAULT)返回一个使用bcrypt加密，并自动添加
了salting的哈希值。验证密码的时候则调用password_verify($password, $hash)。API现在默认使用bcrypt，将来可能会引入其他新的更安全的加密
方式。开发者可以自己调整bcrypt的参数来提高加密强度，可以自己指定salt值等（但是官方不建议这么做）。

## finally

PHP5.5开始支持其他语言的异常处理中常用的finally关键字，开发者可以从此在try和catch块之后，运行指定代码，而无需关心是否有异常抛出，然
后再回到正常执行流，而在此之前，开发者只能在try和catch块中拷贝代码，来完成相关的任务清理工作。比如下面的例子，必须在两个地方
调用releaseResource()：
{% highlight php %}
<?php
function doSomething() {
    $resource = createResource();
    try {
        $result = useResource($resource);
    }
    catch (Exception $e) {
        releaseResource($resource);
        log($e->getMessage());
        exit();
    }
    releaseResource($resource);
    return $result;
}
{% endhighlight %}
有了finally关键字后，就可以删除冗余代码：
{% highlight php %}
<?php
function doSomething() {
    $resource = createResource();
    try {
        $result = useResource($resource);
        return $result;
    }
    catch (Exception $e) {
        log($e->getMessage());
        exit();
    }
    finally {
        releaseResource($resource);
    }
}
{% endhighlight %}
修改后的代码，我们只需在finally块中调用清理函数releaseResource()，无论流程最终是走到try中的return语句，还是到catch中的exit，finally
中的代码都会执行。

## 数组和字符串字面量解引用

现在访问数组的语法中，支持数组和字符串字面量的解引用：
{% highlight php %}
<?php
// array dereferencing - returns 3
echo [1, 3, 5, 7][1];

// string dereferencing - returns "l"
echo "hello"[3];
{% endhighlight %}
这个特性主要是增强了语言的一致性，对我们平时写代码的行为可能影响不大，但是在某些情景下使用还是非常便利的：
{% highlight php %}
<?php
$randomChar = "abcdefg0123456789"[mt_rand(0, 16)];
{% endhighlight %}

## empty()支持函数调用和表达式

empty()这个语言结构开始支持在函数调用和表达式中使用，如empty($object->getProperty())。这样就可以用empty()判断函数返回值，而不
用先把返回值赋值给一个临时变量，然后对临时变量使用empty()。

## 类名解析

从PHP5.3引入命名空间之后，使用它来组织PHP项目中的类结构已经司空见惯，但是要取回带命名空间的全限定类名，却是非常的困难，如：
{% highlight php %}
<?php
use Namespaced\Class\Foo;

$reflection = new ReflectionClass("Foo");
{% endhighlight %}
这个代码将会失败，因为它会从全局命名空间中查找类Foo，而不是指定的命名空间。PHP5.5引入了class关键字，可以通过它获取全限定类名：
{% highlight php %}
<?php
use Namespaced\Class\Foo;

$reflection = new ReflectionClass(Foo::class);
{% endhighlight %}
上面的代码中，Foo:class会解析为"Namespaced\\Class\\Foo"。

## foreach改进

list()语言结构可以便捷地把数组中的值赋给一组变量，如：
{% highlight php %}
<?php
$values = ["sea", "blue"];
list($object, $description) = $values;

// returns "The sea is blue"
echo "The $object is $description";
{% endhighlight %}
现在开始可以在foreach遍历多维数组时，使用list()：
{% highlight php %}
<?php
$data = [
    ["sea", "blue"],
    ["grass", "green"]
];
 
foreach ($data as list($object, $description)) {
    echo "The $object is $description\n";
}

/* Outputs:
The sea is blue
The grass is green
*/
{% endhighlight %}
这个特性使得嵌套数组的遍历变得更加的容易和简洁，并且foreach循环开始支持非标量的值作为迭代器的key，即元素的key可以是字符串和整数之外的
其他类型值。

## 结论

PHP5.5为PHP的高效开发提供了许多改进，除了这些新特性，也修改了大量的bug，具体参考[ChangeLog][1]

原文链接：<http://phpmaster.com/whats-new-in-php-5-5/>

[1]: http://php.net/ChangeLog-5.php#5.5.0 
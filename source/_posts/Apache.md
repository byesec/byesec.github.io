---
title: Apache服务攻防
author: 干 脆 面
tags:
  - 中间件安全
  - "Apache\U0001FAB6"
categories:
  - "安全\U0001F510"
top: 11
abbrlink: 1eb6476e
date: 2022-02-15 21:18:25
---

# Apache服务攻防

## 一、Apache底层解析

### 介绍

Apache(音译为[阿帕奇](https://baike.baidu.com/item/阿帕奇/374191))是世界使用排名第一的Web服务器软件。它可以运行在几乎所有广泛使用的[计算机平台](https://baike.baidu.com/item/计算机平台/2606037)上，由于其[跨平台](https://baike.baidu.com/item/跨平台/8558902)和安全性被广泛使用，是最流行的Web服务器端软件之一。它快速、可靠并且可通过简单的API扩充，将[Perl](https://baike.baidu.com/item/Perl)/[Python](https://baike.baidu.com/item/Python)等[解释器](https://baike.baidu.com/item/解释器/10418965)编译到服务器中。

简而言之，好用！并且能支持基础的HTML、PHP、Perl、Python等语言。

<!-- more -->

### 请求流程

![image-20220207144236732](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220207144236732.png)

浏览器向web服务器发送一个请求，如果请求的是静态资源，那么Apache直接返回给浏览器；如果请求的是动态资源，那么Apache会调用脚本程序php去处理(可能会与数据库交互)，然后在返回给Apache封装成静态资源后返回给浏览器，浏览器将其展示出来。

### PHP架构

![image-20220207154607370](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220207154607370.png)

> 深入理解Zend SAPIs(Zend SAPI Internals)
>
> 参考连接：https://www.laruence.com/2008/08/12/180.html

众所周知，**Apache本身不支持PHP解析的**，结合PHP架构图可知是如何通过SAPI进行通信的：

>#加载php5_module模块
>
>LoadModule php5_module php5apache2_2.dll的路径

>#添加可以执行php的文件类型，让.php文件类型解析为PHP
>
>AddType application/x-httpd-php .php

>#或者以另一种方式（在Apache 2.4.0~2.4.29中默认使用了该方式）
>
><FilesMatch \.php$>
>	SetHandler application/x-httpd-php
></FilesMatch>
>以及
><IfModule dir_module>
>DirectoryIndex index.html index.htm index.php index.phtml 
></Ifmodule>

总结**Apache调用php的流程**为：

**HTTP --> Apache --> php5_module --> sapi --> php**

### 目录结构

常见目录：

bin ------------- 存放常用的命令工具。例如httpd

cgi-bin --------- 存放Linux下常用的命令，例如xxx.sh

conf ----------- Linux的配置相关文件，例如httpd.conf

error ----------- 错误记录

htdocs ----------- 放网站源码

icons ------------- 网站图标

logs ---------------------- 日志

manual -------------------- 手册

modules -------------------- 扩展模块

## 二、Apache攻防解析

### 未知扩展名解析漏洞

#### 1.漏洞描述

apache的解析漏洞依赖于一个特性： **Apache默认一个文件可以有多个以点分割的后缀，当最右边的后缀无法识别（不在mime.types文件内），则继续向左遍历识别，直到识别到合法后缀才进行解析。**

apache认识哪些**扩展名**？--在**/conf/mime.types文件**中有详细的扩展名列表。

而这个特性源于apache的配置。

也就是说如果分别访问存在漏洞的环境下的`a.php.jpg`和`a.php.xxx`。那么`a.php.jpg`会被识别为按照jpg图片进行解析，而`a.php.xxx`才会被当成php脚本进行解析。

#### 2.漏洞复现



##### 复现1:Windows

1.windows下开启phpstudy环境

![image-20220208154418760](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220208154418760.png)

查看phpstudy探针

![image-20220208154523903](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220208154523903.png)

```
Apache/2.4.23 (Win32) OpenSSL/1.0.2j PHP/5.4.45
```

打开配置文件conf/mime.types查看配置

![image-20220208154814792](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220208154814792.png)

建立test.jpg和test.jpg.aa

![image-20220208155301915](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220208155301915.png)

结果显示，当apache从右至左判断后缀时，aa不能被正常解析，向左顺延，被解析为test.jpg

![image-20220208160608528](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220208160608528.png)

访问phpinfo.php.aa

![image-20220208160507613](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220208160507613.png)

![image-20220208160525506](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220208160525506.png)

无法识别后缀aa，被解析为phpinfo.php



PHP结合[Apache](https://so.csdn.net/so/search?q=Apache&spm=1001.2101.3001.7020)的方式有三种：

1. 作为模块：PHP作为Apache的模块，当有php页面的请求时，Apache加载php模块，加载资源并执行php程序，将执行结果返还给客户端。
2. CGI方式(注：CGI是一个协议，可以实现web服务程序和解释器通信)：PHP作为一个独立的功能程序，当有php页面的请求时，Apache通过CGI方式启动一个PHP进程，PHP进程获取资源并执行后，将执行结果返回给Apache，Apache返回给客户端，之后Apache销毁该进程。
3. FastCGI(注：FastCGI是CGI协议的一个扩展，可以实现将PHP作为一个独立进程，由PHP自身实现进程的创建和销毁)：PHP作为一个独立程序，根据配置文件事前启动N个作业进程用来等待处理用户请求(类似于Apache的Prefork模型，规定默认启动进程数、最大启动进程数、最小启动进程数等等)，当有php页面的请求时，Apache通过代理方式使用FastCGI协议将请求发送给PHP进程，并指定php网页文件所在位置(URL)，PHP进程接收后为其分配一个作业进程并执行脚本。FastCGI好处在于可以使PHP服务作为一个独立服务器，通过Socket与httpd通信，可以使服务分离，可扩展性更强，并且无需像CGI一样需要Apache来控制PHP的启动，提高了Apache的工作效率。

>Windows下图文详解PHP三种运行方式（php_mod、cgi、fastcgi）
>
>参考链接：https://www.cnblogs.com/52php/p/5668823.html

切换至php5版本

![image-20220208163300529](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220208163300529.png)

![image-20220208163329261](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220208163329261.png)

##### 复现2:Kali

###### 安装apache

Kali内默认安装了apache

![image-20220209160121069](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209160121069.png)

###### 开启apache

![image-20220209160244713](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209160244713.png)

![image-20220209160340565](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209160340565.png)

###### 安装php

![image-20220209160442518](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209160442518.png)

###### 测试apache+php运行是否正常

在/var/www/html目录下创建index.php和index.php.xxx

内容都如下

![image-20220209160710465](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209160710465.png)

![image-20220209164225076](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209164225076.png)

###### 访问该文件验证

![image-20220209160759614](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209160759614.png)

搭建完毕（mod方式）

###### 分析apache解析漏洞

基于debian的kali linux，起apache配置文件在/etc/apache2/目录下，apache2.conf说apache核心配置文件，鉴于是以模块模式（module）运行的，故要在/etc/apache2/mods-enabled/目录下查找配置

![image-20220209161756087](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209161756087.png)

![image-20220209161950631](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209161950631.png)

```
<FilesMatch ".+\.ph(ar|p|tml)$">
    SetHandler application/x-httpd-php
</FilesMatch>
```

由配置文件内的信息可知：

以phar、php、phtml结尾的文件会被apache当作php文件解析

apache的解析漏洞的根本原因在**$**

**$**在正则表达式中，用于匹配字符串结尾位置

![image-20220209162528727](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209162528727.png)

如果设置了 RegExp 对象的 Multiline 属性，则 **$** 也匹配 '\n' 或 '\r'。-->**换行符**

```
尝试将php7.4.conf中的$换成\.
```

![image-20220209163156692](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209163156692.png)

修改完成后重启apache

![image-20220209163227649](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209163227649.png)

访问index.php.xxx

![image-20220209163911080](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209163911080.png)

结果显示index.php.xxx被当作php文件执行了

**总结利用的条件为**：

1.apache+php的**运行模式为module模式**，且**正则符合条件**（设置了 RegExp 对象的 Multiline 属性，则$会匹配到字符串结尾的换行符）

2.文件扩展名中**包含至少一个.php**

#### 3.修复建议

解决方案一

在httpd.conf或httpd-vhosts.conf中加入以下语句，从而禁止文件名格式为*.php.*的访问权限：

```
<FilesMatch ".(php.|php3.|php4.|php5.)"> 

Order Deny,Allow 

Deny from all 

</FilesMatch>
```

解决方案二

如果需要保留文件名，可以修改程序源代码，替换上传文件名中的“.”为“_”：

```
$filename = str_replace('.', '_', $filename);
```



### AddHandler导致的解析漏洞

#### 1.漏洞描述

(1)apache在解析文件时有一个原则：当碰到不认识的扩展名时，将会从后往前解析，直到遇到认识的扩

展名为止

(2)如果都不认识将会暴露源码。

在apache配置不当的时候就会造成apache解析漏洞。

补充

```
AddType
将给定的文件扩展名映射到指定的内容类型
AddHandler
将文件扩展名映射到指定的处理程序
参考链接：
http://httpd.apache.org/docs/2.0/mod/mod_mime.html#addtype
http://httpd.apache.org/docs/2.0/mod/mod_mime.html#addhandler
```

#### 2.漏洞复现

（在此之前还原上一步的php7.4.conf中的修改，并重启apache服务使配置生效）

在/var/www/html/根目录下新建index.php.jpg

![image-20220209165542919](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209165542919.png)

默认配置下直接访问，无法正常解析

![image-20220209165738188](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209165738188.png)

现在修改配置

进入/etc/apache2/sites-enabled/目录，

新建test.conf添加AddHandler（指定扩展名为 .php 的文件应被application/x-httpd-php来处理。）

并重启apache服务

![image-20220209170636001](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209170636001.png)

访问后结果显示成功被解析为php文件执行

![image-20220209170729429](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220209170729429.png)

**注：只要文件名中出现.php，即使最右边的扩展名（后缀名）是在mime.types文件内，就直接能被解析为php文件**

**罕见的后缀总结**

不仅有php外，还有php3,php4,php5,pht,phtml

例如在php7.4.conf中还有phar

```
<FilesMatch ".+\.ph(ar|p|tml)$">
```

**配置问题总结**

/etc/apache2/apache2.conf

1.如果在内有如下配置

```
<FilesMatch "test.jpg">
    SetHandler application/x-httpd-php
</FilesMatch>
```

则只要访问的文件名为test.jpg，便会以php来解析

2.如果在内有如下配置

```
AddHandler php5-script .php
```

则只要文件名内包含.php，即使文件名为其他后缀（如test.php.jpg）也会以php文件解析

3.如果在内有如下配置

```
AddType application/x-httpd-php .jpg
```

即使扩展名为.jpg，也会按照php文件解析

#### 3.修复建议

##### 解决方案一

在httpd.conf或httpd-vhosts.conf中加入以下语句，从而禁止文件名格式为.php.的访问权限：

```
<FilesMatch ".(php.|php3.|php4.|php5.)"> 
Order Deny,Allow
Deny from all 
</FilesMatch>
```

##### 解决方案二

把配置不当的文件进行修改。



### Apache HTTPD 换行解析漏洞（CVE-2017-15715）

参考链接：

https://www.cnblogs.com/ahtoh/p/13219590.html#autoid-3-0-0

#### 1.漏洞描述

Apache HTTPD是一款HTTP服务器，它可以通过mod_php来运行PHP网页。其2.4.0~2.4.29版本中存在
一个解析漏洞，在解析PHP时，1.php\x0a将被按照PHP后缀进行解析，导致绕过一些服务器的安全策
略。
可以看到这里获取文件名是需要单独post一个name的，因为如果通过如下方式

```
 $_FILES['file']['name'] 
```

获取文件名的话，会把\x0a自动去除，所以这种方式获取文件名就不会造成这个漏洞

```
 $_FILES['file']['name'] 
```

上传一个**后缀末尾包含换行符**的文件，来绕过FilesMatch。**绕过FilesMatch不一定能被PHP解析**。

但可以利用这个漏洞**绕过文件上传黑名单限制**

P牛博客文章：

https://www.leavesongs.com/PENETRATION/apache-cve-2017-15715-vulnerability.html

Apache HTTPD 多后缀解析漏洞

https://github.com/vulhub/vulhub/tree/master/httpd/apache_parsing_vulnerability



#### 2.影响范围

**apache2.4.0~2.4.29版本**

```
1.php\x0a => 1.php
apache通过mod_php来运行脚本，其2.4.0-2.4.29中存在apache换行解析漏洞，在解析php时xxx.php\x0a将被按照php后缀进行解析，导致绕过一些服务器的安全策略。该漏洞属于用户配置不当产生的漏洞，与具体中间件版本无关。
```

此漏洞形成的根本原因在于**.php$**中的 **$** 符号，正则表达式中**$**不仅匹配字符串结尾的位置，也可以匹配**\n**或**\r**

```
git clone https://github.com/vulhub/vulhub.git
```

#### 3.漏洞复现

![image-20220211110012242](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220211110012242.png)

![image-20220211110056422](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220211110056422.png)

访问8080端口

![image-20220211113529488](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220211113529488.png)

index.php内使用了黑名单![image-20220214101031217](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220214101031217.png)

上传抓包

![image-20220214101859863](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220214101859863.png)

直接上传显示失败

![image-20220214101950849](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220214101950849.png)

加入**.**使用**2e**判断位置

![image-20220214112010410](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220214112010410.png)

将**2e**处的值替换为**0a**

![image-20220214112321483](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220214112321483.png)



上传成功

![image-20220214112306769](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220214112306769.png)

访问后发现被解析

![image-20220214112408245](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220214112408245.png)

分析源码

![image-20220214112949395](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220214112949395.png)

使用黑名单过滤php后缀的文件，结尾为.php的文件都是php文件，在正则表达式中匹配输入字符串的结尾的位置。若设置了RegExp对象的Multiline属性，则会匹配**\n**与**\r**

复现过程中在文件尾部添加了**0a**-->0x0a-->**\n**，所以匹配成功了。

补充：

```
1.0x0d \r CR这三者代表是回车，是同一个东西，回车的作用只是移动光标至该行的起始位置 

2.0x0a \n CL这三者代表换行，是同一个东西，换行至下一行行首起始位置
```



#### 4.修复建议

1.升级到最新版本

2.或将上传的文件重命名为为时间戳+随机数+.jpg的格式并禁用上传文件目录执行

### Apache目录遍历漏洞

#### 1.漏洞描述

当客户端访问某一个目录时，Apache服务器默认会寻找index list中的文件，若文件不存在，则页面返回403状态吗及**当前目录下的所有文件**。此行为称为目录遍历。

#### 2.漏洞复现

C:\phpStudy\Apache\conf\httpd.conf文件内的默认配置

![image-20220214165015911](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220214165015911.png)

访问某目录

![image-20220215103209455](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220215103209455.png)



#### 3.漏洞防御

在C:\phpStudy\Apache\conf\httpd.conf文件中将

Options **+Indexes** +FollowSymLinks +ExecCGI

改为

Options **-Indexes** +FollowSymLinks +ExecCGI

```
+Indexes 允许目录浏览
-Indexes 禁止目录浏览
```

修改后再次重启apache后尝试访问

![image-20220215103116822](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220215103116822.png)

![image-20220215103005114](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220215103005114.png)

### Apache SSI远程命令执行漏洞

参考链接：

- https://blog.csdn.net/ploto_cs/article/details/108526605
- https://www.wangan.com/docs/287
- https://www.cnblogs.com/ahtoh/p/13219590.html#autoid-4-0-0

#### 1.漏洞描述

SSI（Server Side Include），通常称为“服务端嵌入”和“服务端包含”，是一种类似于asp的基于服务器的网页制作技术。默认扩展名是`.stm、.shtm和.shtml`。

在测试文件上传漏洞的时候，目标服务器不允许上传`php`后缀文件，如果目标服务器开启了`SSI`与`CGI`的支持，则可以尝试上传一个`shtml`文件，并利用`<!--#exec cmd="id" -->` 语法执行任意命令。

#### 2.影响范围

Apache全版本（支持SSI与CGI）

该漏洞与apache版本和PHP版本无关，属于用户配置不当造成的解析漏洞。

#### 3.漏洞复现

![image-20220215104417116](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220215104417116.png)

SSI是放置在HTML页面中的指令，它可以将动态生成的内容添加到现有的HTML页面，而不必通过CGl程序或其他动态技术来提供整个页面。以上是定义采用在Apache官网对SSI的定义，说白了就是可以在HTML中加入特定的指令，也可以引入其他的页面。开启SSI需要单独配置Apache,可以参考SSI配置。

SSI定义和配置深入了解：
https://httpd.apache.org/docs/2.4/howto/ssi.html

SSI可以完成查看时间、文件修改时间、CGl程序执行结果、执行系统命令、连接数据库等操
作，功能非常强大。
我们要利用的就是SS!执行系统命令的功能，正常的一个包含SS指令的文件，此处文件的后缀取决于Apache的配置，默认为shtml，可以如下内容：

```
<pre>
<!--#exec cmd="id" -->
</pre>
```

![image-20220215104942864](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220215104942864.png)

上传test.shtml

![image-20220215105142729](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220215105142729.png)

访问后成功执行

![image-20220215105203906](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220215105203906.png)

访问test.shtml

#### 4.漏洞防御

可以从上传功能的视角进行漏洞修复；也可以放弃使用SSI

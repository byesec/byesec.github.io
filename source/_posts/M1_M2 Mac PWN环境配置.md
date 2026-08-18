---
title: M1/M2 Pwn环境配置
author: patrik
tags:
  - UTM
  - Pwntools
categories:
  - "安全\U0001F510"
  - "Mac\U0001F468‍\U0001F4BB"
  - "CTF\U0001F6A9"
top: 19
abbrlink: 5aa3009d
date: 2023-07-10 17:00:00
---

## 0、前言

​        今年4月份参加某某杯线下赛的时候，尝试去做pwn的基础题目结果环境报错落败而逃😂，在M1/M2芯片的MacBook上因为arm架构的原因，搭建pwn环境并不是那么顺利，于是回来以后就尝试搞了下相关环境。

​        结合在B站看到的UP主:[Retr0的寄存器](https://space.bilibili.com/442050814)的视频[「零基础向」有趣的Pwn博主的Mac x86虚拟机Pwn环境搭建教程](https://www.bilibili.com/video/BV1HM4y1X7Fy/)，成功使用UTM虚拟机搭建了x86架构下的Pwn环境；然后再结合UP主[BOYcc1](https://space.bilibili.com/2064831648)的视频[pwn环境安装](https://www.bilibili.com/video/BV1Se411g79e/)成功安装了相关工具。

​		在搭建、安装的过程中，顺便截图记录了一下，4月写的，今天整理文件的时候翻到了，顺便传上来，希望能对有需要的师傅提供些帮助。

## 1、虚拟机选择：UTM

[UTM](https://mac.getutm.app)
[https://github.com/utmapp/UTM](https://github.com/utmapp/UTM)

## 2、镜像选择
阿里云镜像站下载utuntu 20.04.5

(这里用的是桌面版的，然后删除了桌面环境。也可以尝试直接取下载服务器版的，能更方便些👀)

[About us](https://mirrors.aliyun.com/ubuntu-releases/20.04/ubuntu-200.04.5-desktop-amd64.iso)
[ubuntu-releases安装包下载_开源镜像站-阿里云](http://mirrors.aliyun.com/ubuntu-releases/)

![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682042561933-c06a9e8a-ed7d-4327-85a8-73a4220f6d9e.png)

## 3、导入UTM
选择模拟CPU
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682043196502-8a22a597-df9a-4c25-a64b-5474d27c45f9.png)

![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682043232929-dc0d518a-a4b8-408b-bec6-8f479394092d.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682043278430-99a390d9-9d4c-4ff8-9554-c47021cffd57.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682043321995-ddb43524-3bc9-4338-8980-c6f060b6741c.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682043330634-696e4307-5ece-4c65-a170-5c6c5da76c26.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682043351668-4f574da4-cf34-44cb-9d42-4dda9f080db0.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682043389948-27ad093a-983d-4a93-afdf-ff5b8147989c.png)
打开刚创建的虚拟机 等待加载至桌面后选择install Ubuntu
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682044801960-8dc25c23-a19e-42ec-9db1-8a8ab3ce2176.png)

![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682044860724-f19163bc-c151-4179-97f2-583486fbaff2.png)

![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682045963758-015141ee-41bd-4a84-b55f-a0383c376258.png)

![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682046039369-c48d75bd-6b97-45cc-b452-885caa3d0db3.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682046096252-33981745-a9a0-4327-81ed-1e253e8aa0a2.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682046236421-aff6e9b7-15a2-4ae8-916c-8d21aac47691.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682046458028-9f212271-cbd1-4d59-a9ca-cacf3c654b02.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682051165486-1c201820-0400-4e87-b67b-d8b51aa37635.png)
安装完成后点击重启
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682051417321-9fe39a62-c761-4068-9fc0-12f429aaa3c8.png)
按提示删除刚才的iso文件后 输入ENTER键 等待进度条读取加载完成后看到图形界面然后关机
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052030688-44d40e26-7a30-4ab9-9b71-241b8279dd0f.png)
关机完成后 清除iso路径后再打开虚拟机 等待
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052138023-802cfe29-8b43-4ec2-9cad-06b03e222a27.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052256828-66983d05-6a4b-446a-a0db-f7dbc80417ba.png)
更新的都不更新 一律跳过/下一步
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052599681-b85e96da-8457-4d71-90de-cd195560e2b6.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052616845-0d309c66-5b52-43ec-9c0c-624646652b24.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052626978-48efac06-438b-489e-bcd1-4cabf1ec5088.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052669051-5091049c-1280-46ad-80ed-2e47586f0a36.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052768311-a600a948-0343-40a1-be74-6c4d5bc4b1b2.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052781076-fc88d190-9c32-4ff8-a399-b95d984e4005.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682052798162-664d3096-78dd-4b33-b8a8-3aeb64aef8a3.png)
移除桌面环境
打开终端
安装git

```python
sudo apt install git
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682053536504-34735c98-4576-4560-90c9-fb0c3194dd12.png)
下载Retr0做好的脚本
脚本链接[https://github.com/DDizzzy79/Reg-Retr0.git](https://github.com/DDizzzy79/Reg-Retr0.git)

```bash
git clone https://github.com/DDizzzy79/Reg-Retr0.git
cd Reg-Retr0/
cd Mac_x86_Pwn_Build_Up/
#换源
sudo ./execute_to_change_apt_source.sh
#更新源
sudo apt update
#为移除GNOME桌面环境做重启前准备
sudo ./execute_to_prepare_for_gnome_remove.sh
sudo reboot
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682053870855-76194fc3-87b8-4f36-b5e2-bf0b03a02064.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682054147118-9ebecac1-2d1a-456b-a843-c21b3e4fc90e.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682054469623-fccfa77d-2dd2-43c8-bdd6-7cbd5c3f9a32.png)

```bash
#输入用户名密码后 进入之前的目录
cd /Desktop/Reg-Retr0/Mac_x86_Pwn_Build_Up/
#需要注意删除前需要修改/etc/default/grub
可以先安装vim 方便使用
sudo apt install vim
需要修改 /etc/default/grub 中的 
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash" 
成 
GRUB_CMDLINE_LINUX_DEFAULT="text"
vim /etc/default/grub
#删除和GNOME桌面环境有关的依赖项
sudo ./execute_to_remove_gnome.sh
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682058433274-bb897502-bb8f-4c4a-af78-5337a57ae2ab.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682058745514-87cc3177-7da2-4657-9955-3b1f606a7f8d.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682058910451-75ebee19-4642-4132-95a3-3d4cd7490b66.png)
安装ssh 方便连接

```bash
sudo apt install openssh-server
sudo systemctl status ssh
sudo apt instlal net-tools
ifconfig
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682059611678-3b03a502-2cb5-4704-b249-6d3d6b7f485f.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682059734346-0392d32e-ccf3-4c78-8ad7-6b52032dff02.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682059818658-6cd0d06b-4257-4ac1-8c51-d757708323a4.png)

## 4、工具安装
ubuntu20自带python版本为3.8.10
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682062620207-a77a8f73-3a52-4fd0-9fa7-3115e5c10314.png)

### pip
```bash
sudo apt-get install python3-pip
```
### pwntools
```bash
pip3 install pwntools -i https://pypi.tuna.tsinghua.edu.cn/simple
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682063217297-d0460f57-0407-46d7-b20d-e54cec1e595e.png)
安装好后终端内输入python3 然后输入

```bash
from pwn import *
```
 如果没有报错就证明安装成功了
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682063254332-f62f7a45-29f7-4fe7-bdcb-6f6077026a78.png)
并且会得到俩个工具 checksec 和 ROPgedget
添加环境变量后即可运行checksec

```bash
#安装pwntools后checksec在~/.local/bin下
#查看当前环境变量
echo $PATH
#在~/.bashrc内添加
export PATH=$PATH:/home/patrik/.local/bin
source ~/.bashrc
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682064017924-6a7d7211-bbfc-4e03-a287-b9ea4c3b6cef.png)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/22799436/1682064971239-2f0f2a77-c588-4de9-8b8a-a1675577b20c.png#averageHue=%23353534&clientId=u7ee99dc7-54f3-4&from=paste&height=154&id=u25a6211d&originHeight=308&originWidth=2468&originalType=binary&ratio=2&rotation=0&showTitle=false&size=94799&status=done&style=none&taskId=u4defa55c-6283-482d-b7f6-6652a23a6b3&title=&width=1234)

### gdb
```bash
mkdir tools
cd tools
```
#### 下载peda
```bash
git clone https://github.com/longld/peda.git
```
#### 下载Pwngdb
```bash
git clone https://github.com/scwuaptx/Pwngdb.git 
```
#### 下载pwndbg
```bash
git clone https://github.com/pwndbg/pwndbg
#若网络存在问题可直接本机下载上传至虚拟机内解压
#sudo apt install unzip
#unzip xxx.zip
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682064535988-be2e72db-bf73-4b27-b647-1ad901e52b99.png)
#### 安装pwndbg
```bash
vim pwndbg/setup.sh
修改源为国内源 添加
-i https://pypi.tuna.tsinghua.edu.cn/simple
./setup.sh
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682065329497-3404fa56-bfbc-4206-88f7-3c4925cdec6a.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682065414434-10b29106-7e68-423d-b7a5-c62c34c682e1.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682065492151-de520ab0-959e-4785-a067-490300b8edce.png)
最后报错 pip重新安装指定的包和版本即可

```bash
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple launchpadlib
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682065977569-21824a53-4684-4242-bbc8-45aeb9e7ee64.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682066482566-295e64ea-c065-4359-8c76-6c007bc6f53d.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682066502035-4226af2a-8f5e-449e-b703-4598d673d151.png)
配置Pwngdb

```bash
cd ..
#tools目录下
cp Pwngdb/.gdbinit ~
vim ~/.gdbinit
#打开文件后文件内容修改如下，这里要注意source ～/pwndgb/gdbinit.py 
#一定要在source ~/Pwngdb/angelheap/gdbinit.py前面，要不然会使用默认配置。

```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682067704125-a6c04d9e-51de-46ce-add1-c951d51d2690.png)
```bash
#source ~/tools/peda/peda.py
source ~/tools/pwndbg/gdbinit.py
source ~/tools/Pwngdb/pwngdb.py
source ~/tools/Pwngdb/angelheap/gdbinit.py

define hook-run
python
import angelheap
angelheap.init_angelheap()
end
end
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682067746018-ea8e6da0-9e4a-4efa-a05c-e40b70c7fb80.png)
### seccomp-tools(查沙盒)
```bash
sudo apt install gcc ruby-dev
sudo gem install seccomp-tools
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682066962932-179fb4b1-cd96-4fd7-8f2e-f0779aa8dd08.png)
### ropper
```bash
pip install ropper
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682067126959-c0c042e2-926b-4495-a1b9-e1b1298cc6e5.png)
### qemu(系统内核题)
```python
sudo apt-get install qemu-user qemu-system
```
### glibc-all-in-one
```bash
git clone https://github.com/matrix1001/glibc-all-in-one.git
cd glibc-all-in-one
vim update_list
改为#!/usr/bin/python3
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682068412615-fe598f59-4a34-41e2-97a1-f08df0a93d47.png)
```bash
#更新最新版本的glibc
./update_list
#查看可下载的glibc
cat list
#根据题目所给的libc,找对应版本的连接器,并下载该连接器
./download 2.23-0ubuntu11.3_amd64
#下载好的libc就在libs目录下
ls libs/2.23-0ubuntu11.3_amd64/
#把ld文件和libc复制到pwn题目录下

#一般来说使用时把标记的ld-2.23.so和libd-2.23.so拷贝在程序目录下即可
#.debug文件用于gdb调试
#在gdb设置setdebugf

```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682068479943-643b620a-99bf-4fec-971c-1e52927c885e.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682070582511-13a7b0e9-f2a6-4b6c-9d79-2514c87cc5b1.png)

### patchelf(修改可执行程序的libc版本)
```bash
git clone https://github.com/NixOS/patchelf.git
cd patchelf
./bootstrap.sh
#autoreconf是需要最小化安装的 直接apt是get不下来的
sudo apt install -y dh-autoreconf
./bootstrap.sh
./configure
make
make check
sudo make install
#使用
#查看题目原来的libc和ld “easyheap”为可执行程序 此处为例子
ldd easyheap
#替换libc
patchelf --replace-needed libc.so.6 ./libc-2.23.so ./easyheap
#设置ld文件
patchelf --set-interpreter ./ld-2.23.so ./easyheap
```
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682070312113-cbf7816d-2f15-4e2a-a899-71a5e6d6beb1.png)

![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682070966680-9984bec2-1339-4036-b488-d6084576e562.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682071283971-1d0060af-00ce-4225-b7b8-2029ebd97161.png)
![image.png](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/10/1682071293835-883fc95b-cbf7-4b73-ae7e-c864d4008d75.png)

### zstd
预防报错（extract脚本在解压的时候用到了zstd解压命令）提前安装
```bash
sudo apt-get install zstd
```

LibcSearcher用来泄露libc库中函数的偏移的库，
one_gadget用来寻找libc库中的execve('/bin/sh', NULL, NULL)可以一个gadget就可以getshell

### 

---
title: M1/M2 Docker内安装awvs
author: patrik
tags:
  - awvs
categories:
  - "安全\U0001F510"
  - "Mac\U0001F468‍\U0001F4BB"
top: 18
abbrlink: fd146a69
date: 2023-07-07 15:40:00
---



## 起因

​	m1/m2安装awvs，因为arm芯片的架构问题，即使再虚拟机内也无法正常使用，可使用docker来安装awvs。记录顺便水一下，相关安装过程。

## 步骤

使用homebrew安装docker

```
brew install —cask docker
```

安装后运行docker 命令行拉取镜像

https://hub.docker.com/r/secfa/awvs

```
docker pull secfa/docker-awvs
```

![image-20230707152840451](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/07/image-20230707152840451.png)

启动下载的镜像映射到本地13443端口（提示不匹配 但不影响功能和使用）

```
docker run -it -d -p 13443:3443 --cap-add LINUX_IMMUTABLE secfa/docker-awvs
```

![image-20230707152900118](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/07/image-20230707152900118.png)

查看是否运行正常

```
docker ps -a
```

![image-20230707153026903](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/07/image-20230707153026903.png)

浏览器访问登录即可

```
https://127.0.0.1:13443
账号密码：
Username:admin@admin.com 
password:Admin123
```

![image-20230707153241676](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/07/image-20230707153241676.png)

![image-20230707153320767](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/07/image-20230707153320767.png)

![image-20230707153406019](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2023/07/07/image-20230707153406019.png)

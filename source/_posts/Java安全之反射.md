---
title: Java安全之反射
author: patrik
tags:
  - Java安全
categories:
  - "安全\U0001F510"
top: 22
abbrlink: a820ff44
date: 2024-02-27 17:00:00
---



# 前情提要

​	重新过一遍Java安全，按P牛的《Java安全漫谈》的PDF和源码从头走了一遍（P牛🐮👍），从把PDF里的知识点和Poc测试的过程重新整理了一下。

​	附：

​	P牛的代码审计星球

​	https://wx.zsxq.com/dweb2/index/group/2212251881

​	配套源码：JavaThings

​	https://github.com/phith0n/JavaThings



# 一、反射篇

`Java安全`可以从`反序列化漏洞`开始说起，`反序列化漏洞`⼜可以从`反射`开始说起。

`反射`是⼤多数语⾔⾥都必不可少的组成部分：
- `对象`可以`通过反射获取他的类`，
- `类`可以`通过反射拿到所有 ⽅法（包括私有）`，拿到的⽅法`可以调⽤`，
总之通过“反射”，我们可以将Java这种静态语⾔附加上`动态特性`。

`动态特性`：“⼀段代码，改变其中的变量，将会导致这段代码产⽣功能性的变化，我称之为动态特性”
PHP本身拥有很多动态特性，所以可以通过“⼀句话⽊⻢马”来执⾏各种功能；`Java虽不像PHP那么灵活，但其提供的“反射”功能，也是可以提供⼀些动态特性`。⽐如，这样⼀段代码，在你不知道传⼊的参数值 的时候，你是不知道他的作⽤是什么的：

```java
public void execute(String className, String methodName) throws Exception { 
	Class clazz = Class.forName(className);
	clazz.getMethod(methodName).invoke(clazz.newInstance()); 
	}
```
上述例子中，演示了几个反射里极为重要的方法：
- 获取类的⽅法： `forName`
- 实例化类对象的⽅法： `newInstance` 
- 获取函数的⽅法： `getMethod` 
- 执⾏函数的⽅法： `invoke`
基本上，这⼏个⽅法包揽了Java安全⾥各种和反射有关的Payload。

`forName`不是获取“类”的唯⼀途径，通常来说我们有如下三种⽅式获取⼀个“类”，也就是`java.lang.Class` 对象：
- obj.getClass()
	如果上下⽂中存在某个类的实例`obj`，那么我们可以直接通过`obj.getClass()`来获取它的类 
- Test.class 
	如果你已经加载了某个类，只是想获取到它的 `java.lang.Class` 对象，那么就直接 拿它的 `class` 属性即可。这个⽅法其实不属于反射。 
- Class.forName 
	如果你知道某个类的名字，想获取到这个类，就可以使⽤ `forName` 来获取

在安全研究中，我们使⽤`反射`的⼀⼤`⽬的`，就是`绕过某些沙盒`。
⽐如，上下⽂中如果只有Integer类型的数字，我们如何获取到可以执⾏命令的Runtime类呢？
也许可以这样（伪代码）：
`1.getClass().forName("java.lang.Runtime")`
SpEL题目WP：http://rui0.cn/archives/1015

`forName`有两个函数重载：
- `Class<?> forName(String name)`
- `Class<?> forName(String name, **boolean** initialize, ClassLoader loader)`
第⼀个就是我们最常⻅见的获取class的⽅式，其实可以理解为第⼆种⽅式的⼀个封装：
```java
Class.forName(className) 
// 等于 
Class.forName(className, true, currentLoader)
```
默认情况下，`forName`的第⼀个参数是类名；第⼆个参数表示是否初始化；第三个参数就是 `ClassLoader`。
`ClassLoader`是什么呢？它就是⼀个“加载器”，告诉Java虚拟机如何加载这个类。关于这个点，后⾯还有很多有趣的漏洞利⽤⽅法，这⾥先不展开说了。Java默认的`ClassLoader`就是根据类名来加载类，这个类名是类完整路径，如`java.lang.Runtime`。
第二个参数`initialize`（是否初始化）常常被人误解,勾陈安全实验室有篇讲反射机制的⽂章(http://www.polaris-lab.com/index.php/archives/450/)
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240227104507403.png)
图中有说“构造函数，初始化时执⾏”，其实在`forName`的时候构造函数并不会执⾏，即使我们设置 `initialize=true` 。

那么这个初始化究竟指什么呢？
可以将这个“初始化”理解为类的初始化。我们先来看看如下这个类：
```java
public class TrainPrint {
    {
        System.out.printf("Empty block initial %s\\n", this.getClass());
    }

    static {
        System.out.printf("Static initial %s\\n", TrainPrint.class);
    }

    public TrainPrint() {
        System.out.printf("Initial %s\\n", this.getClass());
    }
}

```
上述的三个“初始化”⽅法有什么区别，调⽤顺序是什么，在安全上有什么价值？
运行后，⾸先调⽤的是 `static {}` ，其次是 `{}` ，最后是`构造函数`。
`static {}` 就是在“类初始化”的时候调⽤的，⽽ `{}` 中的代码会放在构造函数的 `super()` 后⾯，但在当前构造函数内容的前⾯。
所以说， `forName` 中的 `initialize=true` 其实就是告诉Java虚拟机是否执⾏”类初始化“。

假设我们有如下函数，其中函数的参数name可控：

```java
public void ref(String name) throws Exception {
	Class.forName(name); 
	}
```

我们就可以编写⼀个恶意类，将恶意代码放置在`static{}`中，从⽽执⾏：

```java
import java.lang.Runtime; 
import java.lang.Process;

public class TouchFile {
	static { 
		try { 
			Runtime rt = Runtime.getRuntime(); 
			String[] commands = {"touch", "/tmp/success"}; 
			Process pc = rt.exec(commands); 
			pc.waitFor(); 
		} catch (Exception e) { 
			// do nothing }
		}
	}
}
```
当然，这个恶意类如何带⼊⽬标机器中，可能就涉及到ClassLoader的⼀些利⽤⽅法了。

在正常情况下，除了系统类，如果我们想拿到一个类，需要先 `import` 才能使用。而使用forName就不需要，这样对于我们的攻击者来说就十分有利，我们可以加载任意类。

另外，我们经常在一些源码里看到，类名的部分包含 `$` 符号，比如fastjson在 `checkAutoType` 时候就会先将`$`替换成`.`，`$`的作用是**查找内部类**。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240227110030603.png)

Java的普通类`C1`中支持编写内部类`C2`，而在编译的时候，会生成两个文件：`C1.class`和`C1$C2.class`，我们可以把他们看作两个无关的类，通过`Class.forName("C1$C2")`即可加载这个内部类。获得类以后，我们可以继续使用反射来获取这个类中的属性、方法，也可以实例化这个类，并调用方法。

`class.newInstance()`的作用就是调用这个类的无参构造函数，这个比较好理解。不过，我们有时候 在写漏洞利用方法的时候，会发现使用`newInstance`总是不成功，这时候原因可能是：
1. 你使用的类没有无参构造函数
2. 你使用的类构造函数是私有的
最最最常见的情况就是`java.lang.Runtime`，这个类在我们构造命令执行Payload的时候很常见，但我们不能直接这样来执行命令：
```java
Class clazz = Class.forName("java.lang.Runtime"); 
clazz.getMethod("exec", String.class).invoke(clazz.newInstance(), "id");
```
运行会报错：因为`Runtime`类的构造方法是私有的。
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240227110910469.png)
为什么会有类的构造方法是私有的，难道他不想让用户使用这个类吗？这其实涉及 到很常见的设计模式：“单例模式”。（有时候工厂模式也会写成类似）

比如，对于Web应用来说，数据库连接只需要建立一次，而不是每次用到数据库的时候再新建立一个连接，此时作为开发者你就可以将数据库连接使用的类的构造函数设置为私有，然后编写一个静态方法来获取：
```java
public class TrainDB { 
	private static TrainDB instance = new TrainDB();

	public static TrainDB getInstance() { 
		return instance; 
		}

	private TrainDB() { 
	// 建立连接的代码... 
	}
}
```
这样，只有类初始化的时候会执行一次构造函数，后面只能通过`getInstance`获取这个对象，避免建立多个数据库连接。

回到正题，`Runtime`类就是单例模式，我们只能通过`Runtime.getRuntime()`来获取到`Runtime`对象。我们将上述Payload进行修改即可正常执行命令了：
```java
Class clazz = Class.forName("java.lang.Runtime"); 
clazz.getMethod("exec", String.class).invoke(clazz.getMethod("getRuntime").invoke(clazz),
"calc.exe");
```
这里用到了`getMethod`和`invoke`方法。

`getMethod`的作用是通过反射获取一个类的某个特定的公有方法。而学过Java的同学应该清楚，Java中支持类的重载，我们不能仅通过函数名来确定一个函数。所以，在调用`getMethod`的时候，我们需要传给他你需要获取的函数的参数类型列表。

比如这里的`Runtime.exec`方法有6个重载：
![](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/2024/03/07/image-20240227112002750.png)
我们使用最简单的，也就是第一个，它只有一个参数，类型是String，所以我们使用`getMethod("exec", String.class)`来获取 `Runtime.exec` 方法。

`invoke`的作用是执行方法，它的第一个参数是：
- 如果这个方法是一个普通方法，那么第一个参数是类对象
- 如果这个方法是一个静态方法，那么第一个参数是类
这也比较好理解了，我们正常执行方法是`[1].method([2], [3], [4]...)`,
其实在反射里就是`method.invoke([1], [2], [3], [4]...)`。

所以我们将上述命令执行的Payload分解一下就比较好看懂了。
```java
Class clazz = Class.forName("java.lang.Runtime"); 
Method execMethod = clazz.getMethod("exec", String.class); 
Method getRuntimeMethod = clazz.getMethod("getRuntime"); 
Object runtime = getRuntimeMethod.invoke(clazz); 
execMethod.invoke(runtime, "calc.exe");
```

疑问：
1、如果一个类没有无参构造方法，也没有类似单例模式里的静态方法，我们怎样通过反射实例化该类呢？
需要用到一个新的反射方法 `getConstructor`

和`getMethod`类似，`getConstructor`接收的参数是构造函数列表类型，因为构造函数也支持重载，所以必须用参数列表类型才能唯一确定一个构造函数。

获取到构造函数后，我们使用 `newInstance` 来执行。


比如，我们常用的另一种执行命令的方式ProcessBuilder，我们使用反射来获取其构造函数，然后调用 `start()` 来执行命令：
```java
Class clazz = Class.forName("java.lang.ProcessBuilder"); 
((ProcessBuilder) clazz.getConstructor(List.class).newInstance(Arrays.asList("calc.exe"))).star t();
```

`ProcessBuilder`有两个构造函数：
```java
public ProcessBuilder(List<String> command)
public ProcessBuilder(String... command)
```

上面用到了第一个形式的构造函数，所以`getConstructor`的时候传入的是`List.class`。

但是，我们看到，前面这个Payload用到了Java里的强制类型转换，有时候我们利用漏洞的时候（在表 达式上下文中）是没有这种语法的。所以，我们仍需利用反射来完成这一步。

```java
Class clazz = Class.forName("java.lang.ProcessBuilder"); clazz.getMethod("start").invoke(clazz.getConstructor(List.class).newInstance( Arrays.asList("calc.exe")));
```
通过`getMethod("start")`获取到`start`方法，然后`invoke`执行，`invoke`的第一个参数就是ProcessBuilder Object了。

那么如果要使用`public ProcessBuilder(String... command)`这个构造函数，需要怎样反射执行呢
这又涉及到Java里的可变长参数（varargs）了。正如其他语言一样，Java也支持可变长参数，就是当你定义函数的时候不确定参数数量的时候，可以使用`...`这样的语法来表示“这个函数的参数个数是可变的”。
对于可变长参数，Java其实在编译的时候会编译成一个数组，也就是说，如下这两种写法在底层是等价的（也就不能重载）：
```java
public void hello(String[] names) {} 
public void hello(String...names) {}
```
也由此，如果我们有一个数组，想传给hello函数，只需直接传即可：
```java
String[] names = {"hello", "world"}; 
hello(names);
```
那么对于反射来说，如果要获取的目标函数里包含可变长参数，其实我们认为它是数组就行了。
所以，我们将字符串数组的类`String[].class`传给`getConstructor`，获取`ProcessBuilder`的第二种构造函数：
```java
Class clazz = Class.forName("java.lang.ProcessBuilder"); 
clazz.getConstructor(String[].class)
```
在调用`newInstance`的时候，因为这个函数本身接收的是一个可变长参数，我们传给`ProcessBuilder`的也是一个可变长参数，二者叠加为一个二维数组，所以整个Payload如下：
```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
 ((ProcessBuilder)clazz.getConstructor(String[].class).newInstance(new String[][]{{"calc.exe"}})).start();
```


2、如果一个方法或构造方法是私有方法，我们是否能执行它呢？
涉及到`getDeclared`系列的反射了，与普通的`getMethod`、`getConstructor`区别是：
- `getMethod`系列方法获取的是当前类中所有公共方法，包括从父类继承的方法
- `getDeclaredMethod`系列方法获取的是当前类中“声明”的方法，是实在写在这个类里的，包括私有的方法，但从父类里继承来的就不包含了

`getDeclaredMethod`的具体用法与`getMethod`类似
`getDeclaredConstructor`的具体用法和`getConstructor`类似

举个例子，前文我们说过Runtime这个类的构造函数是私有的，我们需要用`Runtime.getRuntime()`来获取对象。其实现在我们也可以直接用`getDeclaredConstructor`来获取这个私有的构造方法来实例化对象，进而执行命令：
```java
Class clazz = Class.forName("java.lang.Runtime"); 
Constructor m = clazz.getDeclaredConstructor(); 
m.setAccessible(true); 
clazz.getMethod("exec", String.class).invoke(m.newInstance(), "calc.exe");
```
可见，这里使用了一个方法 `setAccessible` ，这个是必须的。我们在获取到一个私有方法后，必须用 `setAccessible` 修改它的作用域，否则仍然不能调用。


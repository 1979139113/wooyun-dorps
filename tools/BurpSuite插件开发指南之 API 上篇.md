# BurpSuite插件开发指南之 API 上篇

0x00 前言
=====

BurpSuite 作为一款 Web 安全测试的利器，得益于其强大的代理，操控数据的功能，在 Web 安全测试过程中，为我们省下了不少时间和精力专注于漏洞的挖掘和测试。更重要的是 BurpSuite 提供了插件开发接口，Web 安全测试人员可以根据实际需求自己开发出提高安全测试效率的插件，虽然 BApp Store 上面已经提供了很多插件，其中也不乏优秀好用的插件，但是作为一名专业的 Web 安全测试人员，有必要熟练掌握 BurpSuite 插件开发技术。  
国内针对 BurpSuite 插件开发的技术文档并不是很多，也不够全面，《BurpSuite 插件开发指南》系列文章作为 BurpSuite 非官方非权威开发指南，希望在这块填补一些空白。文章中难免有纰漏，望读者自行验证并及时指出。

《BurpSuite 插件开发指南》系列文章如下：

*   《BurpSuite插件开发指南之 API 篇》
*   《BurpSuite插件开发指南之 Java 篇》
*   《BurpSuite插件开发指南之 Python 篇》

在第一篇文章中，笔者将会介绍 BurpSuite 所提供的开发接口的功能和参数说明，但是不会对 BurpSuite 本身所提供的功能作太多说明。剩余两篇则会分别使用案例阐述 Java 和 Python 进行插件开发的技术。其中会用到 Java 的 Swing 包进行 GUI 编程。

0x01 BurpSuite 插件开发说明
=====

BurpSuite 插件扩展开发所支持的编程语言有 Java 和 Python，使用这两种开发语言开发的插件在功能上并没有太多区别，使用原生的 Java 开发的插件要比 Python 开发的插件，加载和执行效率更高。所以，笔者推荐使用 Java 作为插件开发的首选语言。

**SDK：**

*   开发文档: BurpSuite 插件开发文档在线地址：[BurpSuite Extension Dev Doc](https://portswigger.net/burp/extender/api/index.html)或者可以在 BurpSuite 程序的**“Extender”**标签下的**“APIs”**子标签里找到。
*   SDK 包: 目前，BurpSuite 官网已经不再提供 SDK 包文件的下载，读者可以从 BurpSuite 程序中导出。
*   JPython：使用 Python 开发插件，需要先安装[JPython](http://www.jython.org/downloads.html)。

注：导出 SDK 包文件操作步骤：**“Extender — APIs － Save interface files（左下角）”**

**开发 IDE：**

*   使用 Java 开发插件的读者可以使用 Eclipse 这款强大的 IDE 。当然也可以使用其他 Java 开发 IDE，如 IDEA。
*   使用 Python 开发插件的读者也可以使用 Eclipse （需要安装 PyDev 插件）作为插件开发的 IDE，或者使用 Notepad++， PyCharm。

注：推荐使用 Eclipse 作为插件开发 IDE, 除了 Java 之外，Eclipse 对 JPython 也有很好的支持。

**辅助工具：**

*   [Swing Inspector](http://www.swinginspector.com/)是一个 Java Swing/AWT 用户界面分析、调试工具。可以快速调试，定位问题，提高开发效率。

**学习资料：**  
BurpSuite 官方并未提供详细的开发文档，只有少量的开发实例，可以[在此找到](https://portswigger.net/burp/extender/)。不过，BurpSuite 官方建立了[插件开发者社区](http://forum.portswigger.net/)和[博客](http://blog.portswigger.net/)，可以帮助开发者解答疑问。

在 乌云Drops 上也有两篇专门介绍 BurpSuite 插件开发的文章，读者可以参考：

*   [BurpSuite 扩展开发[1]-API与HelloWold](http://drops.wooyun.org/papers/3962)[By 园长̇̇̇̇̇](http://drops.wooyun.org/author/%E5%9B%AD%E9%95%BF)
*   [burpsuite扩展开发之Python](http://drops.wooyun.org/tools/5751)[By 路人甲](http://drops.wooyun.org/author/%E8%B7%AF%E4%BA%BA%E7%94%B2)

相较于阅读许多参考资料从零开始编写插件，笔者更推荐在熟悉了开发文档后直接在已有的源码中“照猫画虎”进行插件的开发。现有的插件，读者可以在 BApp Store 中找到，Python 版本的插件可以直接查看源码进行学习，Java 版本的插件可以在反编译后查看其源码进行学习。

注：安装了 BApp Store 中的插件后，会在 BurpSuite 所在目录下的 bapps 文件夹中找到插件文件。

**插件开发“方法论”：**  
插件型的应用程序在设计时就一定会考虑插件的开发，因此，主程序与插件之间必然有一种“约定”，在开发插件的过程中，只需按照程序开发者设计的这种“约定”开发就好了，读者在阅读参考官方开发文档时只需注意以下三点：

*   接口方法功能
*   接口方法入参（参数类型和参数个数）
*   接口方法的返回值（返回值类型）

在关注上述三点以后，就可以把多个接口方法联系在一起，组织起一定的逻辑，这样，开发的思路就顺理成章了。

0x02 API 参考
=====

本节将对 BurpSuite 的官方文档做简要的说明，并给出简单的 Demo code，各个接口具体的使用实例，可以在笔者后续的两篇文章中看到。

### IBurpExtender 接口

**public interface IBurpExtender**

所有的扩展必须实现此接口，实现的类名必须为“BurpExtender”。在 burp 包中，必须申明为 public ,并且必须提供一个默认的构造器。  
此接口实现了以下方法：

```
void registerExtenderCallbacks(IBurpExtenderCallbacks callbacks)

```

此方法将在扩展加载后被调用，它注册了一个**IBurpExtenderCallbacks**接口的实例，**IBurpExtenderCallbacks**接口提供了许多在开发插件过程中常用的一些操作。

参数说明:

*   callbacks 是一个**IBurpExtenderCallbacks**对象。

Demo code：

```
package burp;
public class BurpExtender implements IBurpExtender{
    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){
        // TODO here
    }
}

```

### IBurpExtenderCallbacks

**public interface IBurpExtenderCallbacks**

此接口中实现的方法和字段在插件开发过程中会经常使用到。 Burp Suite 利用此接口向扩展中传递了许多回调方法，这些回调方法可被用于在 Burp 中执行多个操作。当扩展被加载后，Burp 会调用**registerExtenderCallbacks()**方法，并传递一个**IBurpExtenderCallbacks**的实例。扩展插件可以通过这个实例调用很多扩展 Burp 功能必需的方法。如：设置扩展插件的属性，操作 HTTP 请求和响应以及启动其他扫描功能等等。

此接口提供了很多的方法和字段，在此不一一列举，具体的说明可以在 burp SDK 中的**IBurpExtenderCallbacks.java**或[https://portswigger.net/burp/extender/api/burp/IBurpExtenderCallbacks.html](https://portswigger.net/burp/extender/api/burp/IBurpExtenderCallbacks.html)中查看。

Demo code：

```
package burp;
public class BurpExtender implements IBurpExtender{
    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){
        callbacks.setExtensionName("Her0in"); //设置扩展名称为 “Her0in”
    }
}

```

### IContextMenuFactory

**public interface IContextMenuFactory**

Burp 的作者在设计上下文菜单功能中采用了工厂模式的设计模式，扩展可以实现此接口，然后调用**IBurpExtenderCallbacks.registerContextMenuFactory()**注册自定义上下文菜单项的工厂。

此接口提供了如下方法：

```
java.util.List<javax.swing.JMenuItem>   createMenuItems(IContextMenuInvocation invocation)

```

当用户在 Burp 中的任何地方调用一个上下文菜单时，Burp 则会调用这个工厂方法。此方法会根据菜单调用的细节，提供应该被显示在上下文菜单中的任何自定义上下文菜单项。

参数说明：

*   invocation - 一个实现**IMessageEditorTabFactory**接口的对象, 通过此对象可以获取上下文菜单调用的细节。

返回值：  
此工厂方法将会返回需要被显示的自定义菜单项的一个列表（包含子菜单，checkbox 菜单项等等）， 若无菜单项显示，此工厂方法会返回**null**。

Demo code：

```
package burp;

import java.util.ArrayList;
import java.util.List;

import javax.swing.JMenu;
import javax.swing.JMenuItem;

public class BurpExtender implements IBurpExtender, IContextMenuFactory{

    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){

        callbacks.setExtensionName("Her0in");
        callbacks.registerContextMenuFactory(this);
    }


    @Override
    public List<JMenuItem> createMenuItems(final IContextMenuInvocation invocation) {

        List<JMenuItem> listMenuItems = new ArrayList<JMenuItem>();
        //子菜单
        JMenuItem menuItem;
        menuItem = new JMenuItem("子菜单测试");  

        //父级菜单
        JMenu jMenu = new JMenu("Her0in");
        jMenu.add(menuItem);        
        listMenuItems.add(jMenu);
        return listMenuItems;
    }
}

```

### IContextMenuInvocation

**public interface IContextMenuInvocation**

此接口被用于获取当 Burp 调用扩展提供的**IContextMenuFactory**工厂里的上下文菜单时的一些细节，如能获取到调用了扩展提供的自定义上下文菜单的 Burp 工具名称（在**IBurpExtenderCallbacks**中定义）或功能组件的名称（在**IContextMenuInvocation**中定义）。

此接口提供了如下方法：

```
// 此方法可被用于获取本地Java输入事件，并作为上下文菜单调用的触发器
java.awt.event.InputEvent   getInputEvent()

// 此方法被用于获取上下文中被调用的菜单
byte    getInvocationContext()

// 此方法被用于获取用户选中的 Scanner 问题的细节
IScanIssue[]    getSelectedIssues()

// 此方法被用于获取当前显示的或用户选中的 HTTP 请求/响应的细节
IHttpRequestResponse[]  getSelectedMessages()

// 此方法被用于获取用户所选的当前消息的界限（消息需可适用）
int[]   getSelectionBounds()

// 此方法被用于获取调用上下文菜单的 Burp 工具
int getToolFlag()

```

Demo code：

```
package burp;

import java.util.ArrayList;
import java.util.List;

import javax.swing.JMenu;
import javax.swing.JMenuItem;

public class BurpExtender implements IBurpExtender, IContextMenuFactory{

    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){
        callbacks.setExtensionName("Her0in");
        callbacks.registerContextMenuFactory(this);
    }


    @Override
    public List<JMenuItem> createMenuItems(final IContextMenuInvocation invocation) {

        List<JMenuItem> listMenuItems = new ArrayList<JMenuItem>();
        // 菜单只在 REPEATER 工具的右键菜单中显示
        if(invocation.getToolFlag() == IBurpExtenderCallbacks.TOOL_REPEATER){
            //子菜单
            JMenuItem menuItem;
            menuItem = new JMenuItem("子菜单测试");  

            //父级菜单
            JMenu jMenu = new JMenu("Her0in");
            jMenu.add(menuItem);        
            listMenuItems.add(jMenu);
        }
        return listMenuItems;
    }
}

```

### ICookie

**public interface ICookie**

此接口用于获取 HTTP cookie 的一些信息。

```
// 此方法用于获取 Cookie 的域
java.lang.String    getDomain()

// 此方法用于获取 Cookie 的过期时间
java.util.Date  getExpiration()

// 此方法用于获取 Cookie 的名称
java.lang.String    getName()

// 此方法用于获取 Cookie 的路径
java.lang.String    getPath()

// 此方法用于获取 Cookie 的值
java.lang.String    getValue()

```

### IExtensionHelpers

**public interface IExtensionHelpers**

此接口提供了很多常用的辅助方法，扩展可以通过调用**IBurpExtenderCallbacks.getHelpers**获得此接口的实例。

开发插件常用的几个方法如下：

```
// 此方法会添加一个新的参数到 HTTP 请求中，并且会适当更新 Content-Length
byte[]  addParameter(byte[] request, IParameter parameter)

// 此方法用于分析 HTTP 请求信息以便获取到多个键的值
IRequestInfo    analyzeRequest(byte[] request)

// 此方法用于分析 HTTP 响应信息以便获取到多个键的值
IResponseInfo   analyzeResponse(byte[] response)

// 构建包含给定的 HTTP 头部，消息体的 HTTP 消息
byte[]  buildHttpMessage(java.util.List<java.lang.String> headers, byte[] body)

// 对给定的 URL 发起 GET 请求
byte[]  buildHttpRequest(java.net.URL url)

// bytes 到 String 的转换
java.lang.String    bytesToString(byte[] data)

// String 到 bytes 的转
java.lang.String    bytesToString(byte[] data)

```

### IExtensionStateListener

**public interface IExtensionStateListener**

扩展可以实现此接口，然后调用**IBurpExtenderCallbacks.registerExtensionStateListener()**注册一个扩展的状态监听器。在扩展的状态发生改变时，监听器将会收到通知。注意：任何启动后台线程或打开系统资源（如文件或数据库连接）的扩展插件都应该注册一个监听器，并在被卸载后终止线程/关闭资源。

此接口只提供了一个方法：

```
void    extensionUnloaded()

```

在插件被 unload （卸载）时，会调用此方法，可以通过重写此方法，在卸载插件时，做一些善后处理工作。

Demo code：

```
package burp;
import java.io.PrintWriter;

public class BurpExtender implements IBurpExtender, IExtensionStateListener{

    private PrintWriter stdout;

    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){

        this.stdout = new PrintWriter(callbacks.getStdout(), true);

        callbacks.setExtensionName("Her0in");
        // 先注册扩展状态监听器
        callbacks.registerExtensionStateListener(this);
    }

    // 重写 extensionUnloaded 方法
    @Override
    public void extensionUnloaded() {
        // TODO 
        this.stdout.println("extensionUnloaded ...");
    }
}

```

### IHttpListener

**public interface IHttpListener**

扩展可以实现此接口。通过调用**IBurpExtenderCallbacks.registerHttpListener()**注册一个 HTTP 监听器。Burp 里的任何一个工具发起 HTTP 请求或收到 HTTP 响应都会通知此监听器。扩展可以得到这些交互的数据，进行分析和修改。

此接口提供了如下方法：

```
void    processHttpMessage(int toolFlag, boolean messageIsRequest, IHttpRequestResponse messageInfo)

```

如果在开发插件的时候需要获取到所有的 HTTP 数据包，包括通过 Repeater 工具自定义修改的请求，则必须实现此接口，重写该方法。

参数说明：

```
// 指示了发起请求或收到响应的 Burp 工具的 ID，所有的 toolFlag 定义在 IBurpExtenderCallbacks 接口中。
int toolFlag

// 指示该消息是请求消息（值为True）还是响应消息（值为False）
messageIsRequest

// 被处理的消息的详细信息，是一个 IHttpRequestResponse 对象
messageInfo

```

Demo code：

```
package burp;

public class BurpExtender implements IBurpExtender, IHttpListener{

    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){

        callbacks.setExtensionName("Her0in");
        callbacks.registerHttpListener(this);
    }

    @Override
    public void processHttpMessage(int toolFlag, boolean messageIsRequest,
            IHttpRequestResponse messageInfo) {

        // TODO here
    }
}

```

### IHttpRequestResponse

**public interface IHttpRequestResponse**

此接口用于检索和更新有关 HTTP 消息的详细信息。

注意：setter 方法通常只能在消息被被处理之前使用，因为它是一个写操作，因此在只读的上下文中也是不可用的。与响应细节相关的 getter 方法只能用在请求发出后使用。

```
// 获取用户标注的注释信息
java.lang.String    getComment()
// 获取用户标注的高亮信息
java.lang.String    getHighlight()
// 获取请求/响应的 HTTP 服务信息
IHttpService    getHttpService()
// 获取 HTTP 请求信息
byte[]  getRequest()
// 获取 HTTP 响应信息
byte[]  getResponse()
// 更新用户标注的注释信息
void    setComment(java.lang.String comment)
// 更新用户标注的高亮信息
void    setHighlight(java.lang.String color)
// 更新 请求/响应的 HTTP 服务信息 
void    setHttpService(IHttpService httpService)
// 更新 HTTP 请求信息
void    setRequest(byte[] message)
// 更新 HTTP 响应信息
void    setResponse(byte[] message)

```

### IHttpRequestResponsePersisted

**public interface IHttpRequestResponsePersisted extends IHttpRequestResponse**

此接口是**IHttpRequestResponse**接口的一个子接口，该接口用于使用**IBurpExtenderCallbacks.saveBuffersToTempFiles()**将一个IHttpRequestResponse 对象的请求和响应消息保存到临时文件。

此接口只提供了一个方法：

```
void    deleteTempFiles()

```

`注：此方法已经过时了，并且不会执行任何操作。`

### IHttpRequestResponseWithMarkers

**public interface IHttpRequestResponseWithMarkers extends IHttpRequestResponse**

此接口是**IHttpRequestResponse**接口的一个子接口，此接口用于那些已被标记的 IHttpRequestResponse 对象，扩展可以使用**IBurpExtenderCallbacks.applyMarkers()**创建一个此接口的实例，或提供自己的实现。标记可用于各种情况，如指定Intruder 工具的 payload 位置，Scanner 工具的插入点或将 Scanner 工具的一些问题置为高亮。

此接口提供了两个分别操作请求和响应的方法：

```
// 获取带有标记的请求信息的详细信息
java.util.List<int[]>   getRequestMarkers()

// 获取带有标记的请求信息的详细信息
java.util.List<int[]>   getResponseMarkers()

```

这两个方法的返回值均为一个整型数组列表，分别表示请求消息/响应消息标记偏移的索引对。列表中的每一项目都是一个长度为 2 的整型数组（int[2](http://www.jython.org/downloads.html)）包含标记开始和结束的偏移量。如果没有定义请求/响应标记，返回 null。

### IHttpService

**public interface IHttpService**

此接口用于提供关于 HTTP 服务信息的细节。

此接口提供了如下方法：

```
// 返回 HTTP 服务信息的主机名或 IP 地址
java.lang.String    getHost()

// 返回 HTTP 服务信息的端口
int getPort()

// 返回 HTTP 服务信息的协议
java.lang.String    getProtocol()

```

Demo code：

```
package burp;

import java.io.PrintWriter;

public class BurpExtender implements IBurpExtender, IHttpListener{

    private PrintWriter stdout;
    public IBurpExtenderCallbacks iCallbacks;

    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){

        this.stdout = new PrintWriter(callbacks.getStdout(), true); 
        callbacks.setExtensionName("Her0in");
        callbacks.registerHttpListener(this);
    }

    @Override
    public void processHttpMessage(int toolFlag, boolean messageIsRequest,
            IHttpRequestResponse messageInfo) {

        IHttpService iHttpService = messageInfo.getHttpService();

        this.stdout.println(iHttpService.getHost());
        this.stdout.println(iHttpService.getPort());
        this.stdout.println(iHttpService.getProtocol());
    }
}

```

### IInterceptedProxyMessage

**public interface IInterceptedProxyMessage**

注：  
在 BurpSuite 的 Proxy 工具下的 Options 标签里有几个自定义消息拦截和响应的功能。读者可以在熟悉了这几个功能后，再了解此接口的作用。

此接口不能被扩展实现，它表示了已被 Burp 代理拦截的 HTTP 消息。扩展可以利用此接口注册一个 IProxyListener 以便接收代理消息的细节。

此接口提供了以下方法：

```
// 获取被拦截的请求消息的客户端 IP 地址
java.net.InetAddress    getClientIpAddress()

// 获取当前定义的拦截操作类型，具体的类型可以在本接口中看到
int getInterceptAction()

// 获取 Burp Proxy 处理拦截消息监听器的名称
java.lang.String    getListenerInterface()

// 获取被拦截的消息的详细信息
IHttpRequestResponse    getMessageInfo()

// 获取请求/响应消息的唯一引用号
int getMessageReference()

// 设置更新拦截操作
void    setInterceptAction(int interceptAction)

```

Demo code：

```
package burp;

import java.io.PrintWriter;

public class BurpExtender implements IBurpExtender, IProxyListener{

    private PrintWriter stdout;

    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){
        callbacks.setExtensionName("Her0in");
        this.stdout = new PrintWriter(callbacks.getStdout(), true);
        callbacks.registerProxyListener(this);
    }

    @Override
    public void processProxyMessage(boolean messageIsRequest,
            IInterceptedProxyMessage message) {
        // 只操作请求消息
        if(messageIsRequest){
            // 获取并打印出客户端 IP
            this.stdout.println(message.getClientIpAddress());
            // Drop 掉所有请求
            message.setInterceptAction(IInterceptedProxyMessage.ACTION_DROP);
            // TODO here
        }       
    }
}

```

### IIntruderAttack

**public interface IIntruderAttack**

此接口用于操控 Intruder 工具的攻击详情。

此接口提供了以下方法：

```
// 获取攻击中的 HTTP 服务信息
IHttpService    getHttpService()

// 获取攻击中的请求模版
byte[]  getRequestTemplate()

```

### IIntruderPayloadGenerator

**public interface IIntruderPayloadGenerator**

此接口被用于自定义 Intruder 工具的 payload 生成器。当需要发起一次新的 Intruder 攻击时，扩展需要注册一个**IIntruderPayloadGeneratorFactory**工厂并且必须返回此接口的一个新的实例。此接口会将当前插件注册为一个 Intruder 工具的 payload 生成器。

```
此接口提供了如下方法：  
// 此方法由 Burp 调用，用于获取下一个 payload 的值
byte[]  getNextPayload(byte[] baseValue)

// 此方法由 Burp 调用，用于决定 payload 生成器是否能够提供更多 payload
boolean hasMorePayloads()

// 此方法由 Burp 调用，用于重置 payload 生成器的状态，这将导致下一次调用 getNextPayload() 方法时会返回第一条 payload
void    reset()

```

Demo code:

```
package burp;

public class BurpExtender implements IBurpExtender, IIntruderPayloadGeneratorFactory{

    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){
        callbacks.setExtensionName("Her0in");
        // 将当前插件注册为一个 Intruder 工具的 payload 生成器
        callbacks.registerIntruderPayloadGeneratorFactory(this);
    }

    @Override
    public String getGeneratorName() {
        // 设置 payload 生成器名称
        return "自定义 payload 生成器";
    }

    @Override
    public IIntruderPayloadGenerator createNewInstance(IIntruderAttack attack) {
        // 返回一个新的 payload 生成器的实例
        return new IntruderPayloadGenerator();
    }

     // 实现 IIntruderPayloadGenerator 接口，此接口提供的方法是由 Burp 来调用的
    class IntruderPayloadGenerator implements IIntruderPayloadGenerator{

        @Override
        public boolean hasMorePayloads() {
            // TODO here
            return false;
        }

        @Override
        public byte[] getNextPayload(byte[] baseValue) {
            // TODO here
            return null;
        }

        @Override
        public void reset() {
            // TODO here

        }
    }
}

```

在 Burp 加载了上述插件后，可以按照下图标红的步骤操作，即可看到自定义的 payload 生成器：

![p1](http://drops.javaweb.org/uploads/images/ff05a352cda7e61c056f2a577c00679499a8a0f4.jpg)

### IIntruderPayloadGeneratorFactory

**public interface IIntruderPayloadGeneratorFactory**

扩展可以实现此接口，并且可以调用**IBurpExtenderCallbacks.registerIntruderPayloadGeneratorFactory()**注册一个自定义的 Intruder 工具的 payload 生成器。

该接口提供了以下方法：

```
// 此方法由 Burp 调用，用于创建一个 payload 生成器的新实例。当用户发动一次 Intruder 攻击时，将会使用该方法返回的 payload 生成器的实例
IIntruderPayloadGenerator   createNewInstance(IIntruderAttack attack)

// 此方法由 Burp 调用，用于获取 payload 生成器的名称
java.lang.String    getGeneratorName()

```

### IIntruderPayloadProcessor

**public interface IIntruderPayloadProcessor**

扩展可以实现此接口，并且可以调用**IBurpExtenderCallbacks.registerIntruderPayloadProcessor()**注册一个自定义 Intruder 工具的 payload 的处理器。此接口会将当前插件注册为一个 Intruder 工具的 payload 处理器。

该接口提供了以下方法：

```
// 此方法由 Burp 调用，用于获取 payload 处理器的名称
java.lang.String    getProcessorName()

// 此方法由 Burp 调用，当处理器每次应用 payload 到一次 Intruder 攻击时,Burp 都会调用一次此方法
byte[]  processPayload(byte[] currentPayload, byte[] originalPayload, byte[] baseValue)

// processPayload 方法说明：

```

参数说明：

*   currentPayload - 当前已被处理过的 payload 的值
*   originalPayload - 在应用处理规则之前的 payload 的原始值
*   baseValue - payload 位置的基准值，将用当前已被处理过的 payload 替代

返回值:  
返回已被处理过的 payload 的值。 如果返回 null 意味着当前的 payload 将被跳过,并且此次攻击将被直接移动到下一个 payload 。

Demo code：

```
package burp;

public class BurpExtender implements IBurpExtender, IIntruderPayloadProcessor{

    @Override
    public void registerExtenderCallbacks(final IBurpExtenderCallbacks callbacks){
        callbacks.setExtensionName("Her0in");
        // 将当前插件注册为一个 Intruder 工具的 payload 处理器
        callbacks.registerIntruderPayloadProcessor(this);
    }

    // 此方法由 Burp 调用
    @Override
    public String getProcessorName() {
        // 设置自定义 payload 处理器的名称
        return "自定义 payload 处理器";
    }

    // 此方法由 Burp 调用，且会在每次使用一个 payload 发动攻击时都会调用一次此方法
    @Override
    public byte[] processPayload(byte[] currentPayload, byte[] originalPayload,
            byte[] baseValue) {
        // TODO here
        return null;
    }
}

```

在 Burp 加载了上述插件后，可以按照下图标红的步骤操作，即可看到自定义的 payload 生成器：

![p2](http://drops.javaweb.org/uploads/images/13d744b01e250b5d3e5622330cd4f15ce8043740.jpg)

### IMenuItemHandler

**public interface IMenuItemHandler**

`此接口已过时，不推荐再使用，请使用IContextMenuFactory代替。`

扩展可以实现此接口，并且通过调用**IBurpExtenderCallbacks.registerMenuItem()**方法注册一个自定义的上下文菜单项。

此接口提供了如下方法：

```
// 当用户单击一个已经在 Burp 中注册过的上下文菜单项时 Burp 会调用一次此方法。不过，此方法已经过时
void    menuItemClicked(java.lang.String menuItemCaption, IHttpRequestResponse[] messageInfo)  

```

**本文由 Her0in 原创并首发于乌云drops，转载请注明出处**
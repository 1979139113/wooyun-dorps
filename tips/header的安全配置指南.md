# header的安全配置指南

from:[https://blog.veracode.com/2014/03/guidelines-for-setting-security-headers/](https://blog.veracode.com/2014/03/guidelines-for-setting-security-headers/)

0x00 背景
-------

* * *

在统计了Alexa top 100万网站的header安全分析之后（2012年11月 - 2013年3月 - 2013年11月），我们发现其实如何正确的设置一个header并不是一件容易的事情。尽管有数不胜数的网站会使用大量有关安全方面的header，但并没有一个像样的平台能够为开发者们提供必要的信息，以辨别那些常见的错误设置。或者说，即使这些安全方面的header设置正确了，也没有一个平台能够为开发者提供一个系统的测试方法，用来测试正确与否。这些header如果设置错误了不仅会产生安全的假象，甚至会对网站的安全产生威胁。veracode认为安全性header是网络防护中非常重要的一环，并且他希望让开发者们能够简捷、正确地设置站点。如果您对某一header或设置有任何疑问，我们有极好的资源能够追踪到浏览器支持情况。

0x01 细节
-------

* * *

### 1. X-XSS-Protection

**目的**

这个header主要是用来防止浏览器中的反射性xss。现在，只有IE，chrome和safari（webkit）支持这个header。

**正确的设置**

```
0 – 关闭对浏览器的xss防护  
1 – 开启xss防护  
1; mode=block – 开启xss防护并通知浏览器阻止而不是过滤用户注入的脚本。  
1; report=http://site.com/report – 这个只有chrome和webkit内核的浏览器支持，这种模式告诉浏览器当发现疑似xss攻击的时候就将这部分数据post到指定地址。  

```

**通常不正确的设置**

```
0; mode=block; – 记住当配置为0的时候，即使加了mode=block选项也是没有效果的。需要指出的是，chrome在发现这种错误的配置后还是会开启xss防护。  
1 mode=block; – 数字和选项之间必须是用分号分割，逗号和空格都是错误的。但是这种错误配置情况下，IE和chrome还是默认会清洗xss攻击，但是不会阻拦。 

```

**如何检测**

如果过滤器检测或阻拦了一个反射性xss以后,IE会弹出一个对话框。当设置为1时，chrome会隐藏对反射性xss的输出。如果是设置为 1; mode=block ,那么chrome会直接将user-agent置为一个空值:, URL  这种形式。

**参考文献**

Post from Microsoft on the X-XSS-Protection Header  
Chromium X-XSS-Protection Header Parsing Source  
Discussion of report format in WebKit bugzilla

### 2. X-Content-Type-Options

**目的**

这个header主要用来防止在IE9、chrome和safari中的MIME类型混淆攻击。firefox目前对此还存在争议。通常浏览器可以通过嗅探内容本身的方法来决定它是什么类型，而不是看响应中的content-type值。通过设置 X-Content-Type-Options：如果content-type和期望的类型匹配，则不需要嗅探，只能从外部加载确定类型的资源。举个例子，如果加载了一个样式表，那么资源的MIME类型只能是text/css，对于IE中的脚本资源，以下的内容类型是有效的：

```
application/ecmascript  
application/javascript  
application/x-javascript  
text/ecmascript  
text/javascript  
text/jscript  
text/x-javascript  
text/vbs  
text/vbscript  

```

对于chrome，则支持下面的MIME 类型：

```
text/javascript  
text/ecmascript  
application/javascript  
application/ecmascript  
application/x-javascript  
text/javascript1.1  
text/javascript1.2  
text/javascript1.3  
text/jscript  
text/live script

```

**正确的设置**

```
nosniff – 这个是唯一正确的设置，必须这样。  

```

通常不正确的设置

```
‘nosniff’ – 引号是不允许的  
: nosniff – 冒号也是错误的  

```

**如何检测**

在IE和chrome中打开开发者工具，在控制台中观察配置了nosniff和没有配置nosniff的输出有啥区别。

**参考文献**

Microsoft Post on Reducing MIME type security risks  
Chromium Source for parsing nosniff from response  
Chromium Source list of JS MIME types  
MIME Sniffing Living Standard

### 3. X-Frame-Options

**目的**

这个header主要用来配置哪些网站可以通过frame来加载资源。它主要是用来防止UI redressing 补偿样式攻击。IE8和firefox 18以后的版本都开始支持ALLOW-FROM。chrome和safari都不支持ALLOW-FROM，但是WebKit已经在研究这个了。

**正确的设置**

```
DENY – 禁止所有的资源（本地或远程）试图通过frame来加载其他也支持X-Frame-Options 的资源。  
SAMEORIGIN – 只允许遵守同源策略的资源（和站点同源）通过frame加载那些受保护的资源。  
ALLOW-FROM http://www.example.com – 允许指定的资源（必须带上协议http或者https）通过frame来加载受保护的资源。这个配置只在IE和firefox下面有效。其他浏览器则默认允许任何源的资源（在X-Frame-Options没设置的情况下）。 

```

**通常不正确的设置**

```
ALLOW FROM http://example.com – ALLOW和FROM 之间只能通过连字符来连接，空格是错误的。  
ALLOW-FROM example.com – ALLOW-FROM选项后面必须跟上一个URI而且要有明确的协议（http或者https）  

```

**如何检测**

可以通过访问test cases 来查看各种各样的选项 和浏览器对这些frame中的资源的响应。

**参考文献**

X-Frame-Options RFC  
Combating ClickJacking With X-Frame-Options

### 4. Strict-Transport-Security

**目的**

Strict Transport Security (STS) 是用来配置浏览器和服务器之间安全的通信。它主要是用来防止中间人攻击，因为它强制所有的通信都走TLS。目前IE还不支持 STS头。需要注意的是，在普通的http请求中配置STS是没有作用的，因为攻击者很容易就能更改这些值。为了防止这样的现象发生，很多浏览器内置了一个配置了STS的站点list。  
   
**正确的设置**   
注意下面的值必须在https中才有效，如果是在http中配置会没有效果。  
 

```
max-age=31536000 – 告诉浏览器将域名缓存到STS list里面，时间是一年。  
max-age=31536000; includeSubDomains – 告诉浏览器将域名缓存到STS list里面并且包含所有的子域名，时间是一年。  
max-age=0 – 告诉浏览器移除在STS缓存里的域名，或者不保存此域名。  

```

**通常不正确的设置**

```
直接将includeSubDomains设置为 https://www.example.com ，但是用户依然可以通过 http://example.com 来访问此站点。如果example.com 并没有跳转到 https://example.com 并设置 STS header，那么访问 http://www.example.com 就会直接被浏览器重定向到 https://www.example.com 。  
max-age=60 – 这个只设置域名保存时间为60秒。这个时间太短了，可能并不能很好的保护用户，可以尝试先通过http来访问站点，这样可以缩短传输时间。  
max-age=31536000 includeSubDomains – max-age 和 includeSubDomains 直接必须用分号分割。这种情况下，即使max-age的值设置的没有问题，chrome也不会将此站点保存到STS缓存中。  
max-age=31536000, includeSubDomains – 同上面情况一样。  
max-age=0 – 尽管这样在技术上是没有问题的，但是很多站点可能在处理起来会出差错，因为0可能意味着永远不过期。 

```

**如何检测**

判断一个主机是否在你的STS缓存中，chrome可以通过访问chrome://net-internals/#hsts，首先，通过域名请求选项来确认此域名是否在你的STS缓存中。然后，通过https访问这个网站，尝试再次请求返回的STS头，来决定是否添加正确。

**参考文献**

Strict Transport Security RFC6797  
Wikipedia page on Strict Transport Security (with examples)

### 5. Public-Key-Pins (起草中)

**目的**

有关这个header的详细描述还在起草中，但是它已经有了很清晰的安全作用，所以我还是把它放在了这个list里面。Public-Key-Pins (PKP)的目的主要是允许网站经营者提供一个哈希过的公共密钥存储在用户的浏览器缓存里。跟Strict-Transport-Security功能相似的是，它能保护用户免遭中间人攻击。这个header可能包含多层的哈希运算，比如pin-sha256=base64(sha256(SPKI))，具体是先将 X.509 证书下的Subject Public Key Info (SPKI) 做sha256哈希运算，然后再做base64编码。然而，这些规定有可能更改，例如有人指出，在引号中封装哈希是无效的，而且在33版本的chrome中也不会保存pkp的哈希到缓存中。

这个header和 STS的作用很像，因为它规定了最大子域名的数量。此外，pkp还提供了一个Public-Key-Pins-Report-Only 头用来报告异常，但是不会强制阻塞证书信息。当然，这些chrome都是不支持的。

**正确的设置**

```
max-age=3000; pin-sha256=”d6qzRu9zOECb90Uez27xWltNsj0e1Md7GkYYkVoZWmM=”; – 规定此站点有3000秒的时间来对x.509证书项目中的公共密钥信息（引号里面的内容）做sha256哈希运算再做base64编码。  
max-age=3000; pin-sha256=”d6qzRu9zOECb90Uez27xWltNsj0e1Md7GkYYkVoZWmM=”; report-uri=”http://example.com/pkp-report” – 同上面一样，区别是可以报告异常。  

```

**通常不正确的设置**

```
max-age=3000; pin-sha256=d6qzRu9zOECb90Uez27xWltNsj0e1Md7GkYYkVoZWmM=; – Not encapsulating the hash value in quotes leads to Chrome 33 not adding the keys to the PKP cache. This mistake was observed in all but one of the four sites that returned this or the report-only header response.没有添加引号，这样的话chrome不会将这个key添加到PKP缓存中。我们的调查中发现有四分之一的网站存在此问题。

```

**如何检测**

如果站点成功的将pkp哈希存入了客户端缓存，那么使用和Strict-Transport-Security （STS）相同的方法查看pubkey_hashes 信息就可以了。

**参考文献**

Public-Key-Pins draft specification  
Chrome issue tracking the implementation of Public-Key-Pins  
Chromium source code for processing Public-Key-Pins header

### 6. Access-Control-Allow-Origin

**目的**

Access-Control-Allow-Origin是从Cross Origin Resource Sharing (CORS)中分离出来的。这个header是决定哪些网站可以访问资源，通过定义一个通配符来决定是单一的网站还是所有网站可以访问我们的资源。需要注意的是，如果定义了通配符，那么 Access-Control-Allow-Credentials选项就无效了，而且user-agent的cookies不会在请求里发送。

**正确的设置**

```
* – 通配符允许任何远程资源来访问含有Access-Control-Allow-Origin 的内容。  
http://www.example.com – 只允许特定站点才能访问(http://[host], 或者 https://[host]) 

```

**通常不正确的设置**

```
http://example.com, http://web2.example.com – 多个站点是不支持的，只能配置一个站点。  
*.example.com – 只允许单一的站点  
http://*.example.com – 同上面一样  

```

**如何检测**

很容易就能确定这个header是否被设置的正确，因为如果设置错误的话，CORS请求就会接收不到数据。

**参考文献**

W3C CORS specification  
Content-Security-Policy 1.0

**目的**

csp是一些指令的集合，可以用来限制页面加载各种各样的资源。目前，IE浏览器只支持CSP的一部分，而且仅支持X-Content-Security-Policy header。 相比较而言，Chrome和Firefox则支持CSP的1.0版本，csp的1.1版本则还在开发中。通过恰当的配置csp，可以防止站点遭受很多类型的攻击，譬如xss和UI补偿等相关问题。

csp总共有10种配置形式，每一种都可以用来限制站点何时加载和加载何种类型的资源。他们分别是：

default-src：这种形式默认设置为 script-src, object-src, style-src, img-src, media-src, frame-src, font-src和connect-src.如果上述设置一个都没有的话，user-agent就会被用来作为default-src的值。

script-src：这种形式也有两个附加的设置。

```
1、unsafe-inline：允许资源执行脚本。举个例子，on事件的值会被编码到html代码中，或者脚本元素的text的内容会被混入到受保护的资源中。  
2、unsafe-eval：允许资源动态的执行函数，比如eval，setTimeout, setInterval, new等函数。

```

object-src – 决定从哪里加载和执行插件。  
style-src – 决定从哪里加载css和样式标记。  
img-src – 决定从哪里加载图片。  
media-src – 决定从哪里加载视频和音频资源。  
frame-src – 决定哪里的frames 可以被嵌入。  
font-src – 决定从哪里加载字体。  
connect-src – 限制在 XMLHttpRequest, WebSocket 和 EventSource 中可以使用哪些类型的资源。  
sandbox – 这是一个可选形式，它决定了沙盒的策略，如何将内容嵌入到沙盒中以保证安全。

当这个策略被特定的url违反了，我们也可以用报告地址直接发送报告。这样做有利于debug和当攻击发生时通知我们。

此外， 我们可以定义Content-Security-Policy-Report-Only 的 header不强制遵守csp，但是会发送潜在的威胁到一个报告地址。它遵守和csp一样的语法和规则。

**正确的设置**

```
View cspplayground.com compliant examples  

```

通常不正确的设置：

```
View cspplayground.com violation examples  

```

**如何检测**

在chrome和firefox中打开开发者工具或者firebug，在控制台查看是否有潜在的威胁。

**参考文献**

Content Security Policy 1.0 Specification  
Content Security Policy 1.1 Working Draft Specification  
cspplayground.com’s excellent collection of resources for CSP  
Sublime Text 2 Plugin for validating HTML/JS while you code your site

关于Alexa排名top 100万网站的安全header的研究
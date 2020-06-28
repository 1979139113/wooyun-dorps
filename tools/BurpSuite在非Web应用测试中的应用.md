# BurpSuite在非Web应用测试中的应用

0x00 前言
=====

Burp不仅仅能在Web应用的测试中使用。我也在移动端和传统客户端测试时经常使用。对于采用HTTP方法发送数据的应用，Burp是你的最佳选择。

我要写下一系列在我工作中给我帮助的那些Burp的提示和技巧。写出来为了与大家分享，也为了留着备忘。

在撰写此文时Burp Pro的最新版本是1.6.39，大多数情况下应用的Burp Free的最新版本是1.6.32。我最初使用的是Burp 1.5，自此版本之后Burp并没有太大改变。

当我最初准备写时，我没有想到Burp有这么多可写的。所以我把它拆分成多个部分。请注意，这篇文章不是在写Web应用测试，所以我会跳过一些功能不做介绍。如果你有任何好用的提示或用例，一如既往的，我们欢迎反馈。

0x01 拦截
=====

Burp支持请求/响应包的拦截和修改。你可以在Proxy > Options 中配置这些设置。

1.1 拦截响应
--------

有时候为了手动修改你不得不拦截响应数据包。设置在Proxy > Options > Intercept responses based。将第一条规则的复选框取消，否则二进制payload有可能不被拦截。

![拦截请求](http://drops.javaweb.org/uploads/images/5fd1282f6f74c9deff27fb6060bd9bb38b2969ca.jpg)

1.2 拦截请求/响应规则
-------------

Burp支持为拦截请求/响应数据包制定规则。当有多重流量经过Burp时，你只想将某些特定节点的流量拦截下来时，这将非常有用。在Proxy > Options 中可以看到规则Intercept Client / Server Requests。这个预定义的规则只能拦截域内的请求。你也可以添加你自己的规则，规则支持正则表达式来匹配内容和包头。

![拦截规则](http://drops.javaweb.org/uploads/images/0e8851bde742082ca7cc9e9a0a646bdcbe868085.jpg)

1.3 匹配和替换
---------

你可以在Proxy > Options > Match and Replace 中找到匹配与替换。这意味着你可以在请求/响应数据包中按照你的要求进行替换。我通常用用来替换User-Agent。另一种方法不需要打补丁就能绕过客户端控件在响应数据包中自动进行更改。例如，如果服务器响应登录是否成功，我定义了一条规则来将登录失败响应修改为成功响应，来绕过登录（这只有在服务器不关心你的登录是否成功时起作用）。

![匹配和替换规则](http://drops.javaweb.org/uploads/images/ab593beaa31afb62dbedebffab6a8de1fa78a0c0.jpg)

1.4 SSL流量
---------

这是一个Burp被低估的功能，在Proxy > Options > SSL Pass Through中可以找到。

Burp这个功能的实现，不是像中间人攻击一样，而只是像一个非终端的TLS代理。

如果你想开启代理，但是代理却没有工作。你可以把这个节点加入 SSL Pass Through中，看这个问题是否与Burp有关。

传统客户端会在不同的节点混合使用HTTP和非HTTP协议的流量是很常见的。Burp可以在非HTTP连接中架起中间人，可以静默的丢包或者改包。这将会导致应用程序出现故障。首先应该识别终端节点，并且把它们添加进SSL Pass Through 中。可以看一个[实践案例](http://parsiya.net/blog/2015-10-09-proxying-hipchat-part-2-so-you-think-you-can-use-burp/#3-burp-s-ssl-pass-through:4e05e28f6fdfa8cb6420fb7940d5416d)

你可以使用这些功能使Burp变成一个快速简单的端口转换器。假设你想要连接一个客户端，这个客户端通过1234端口发送给远端服务器所监听的5678端口。如果你不希望自己写代码实现，又不希望使用其他工具来重定向端口。把Burp放在1234端口上开代理，使用hosts文件或其他操作系统特有的文件来重定向这个终端节点到本地。在Burp中你可以通过设置Request Handling 来让代理重定向全部流量到某个终端节点的不同端口。将这个终端节点添加进SSL Pass Through 中。

1.5 响应修改选项
----------

这些中的大多数可能只是对于Web应用有效。

Convert HTTPS links to HTTP 和 Remove secure flag from cookies 和Force use of SSL都是Request Handling中的好工具。如果在应用或者浏览器和Burp之间禁用TLS，设置为Secure的Cookie将不会再传输，app也会停止工作。当使用Cookie设置响应头时，Burp可以移除Secure标识。

![响应修改](http://drops.javaweb.org/uploads/images/50c872cd8b84217687e9bc43d3ab826fa8ba57ae.jpg)

1.6 在启动时禁用拦截和混淆
---------------

我运行了Burp然后设置好了代理，运行应用却没有反应。然后我意识到启动Burp时默认已经开启了拦截。

Proxy > Options > Scroll all the way to the bottom > Under Miscellaneous > Enable interception at startup > Always disable.

![拦截启动设置](http://drops.javaweb.org/uploads/images/274db70482d06b01a471d3a08bc286449204400e.jpg)

0x02 代理监听
=====

Burp监听之前流量经过的那个端口。默认为127.0.0.1:8080，你也可以自行设置。你可以让新的代理监听其他网口或者全部网口。唯一的限制是另一个程序就不能使用被选定的网口的这个端口了。

代理监听可以在Proxy > Option > Proxy Listeners设置。

![代理监听](http://drops.javaweb.org/uploads/images/26b440baa12eaace63df704aba179a6aff158987.jpg)

2.1 绑定
------

添加一个新的监听很容易，只需要点击Add即可。本地环回是127.0.0.1或localhost。如果你想让Burp监听另一个网口，就可以在这里进行选择。如果想代理移动设备这就非常有用了。在这种情况下，我会创建一个监听为0.0.0.0或者选择All interfaces，这些和移动设备共享的网口（例如Windows hostednetwork）

![代理绑定](http://drops.javaweb.org/uploads/images/7fb4e32365055446005754bd846618f17a7d4a7c.jpg)

我们可以在Import/export CA certificate 或Regenerate CA certificate来导入/导出一个新的呃Burp的根证书。更多信息请[参考](http://parsiya.net/blog/2016-02-21-installing-burp-certificate-authority-in-windows-certificate-store/)。如果你重新生成根证书，你必须在操作系统或浏览器的证书存储区使用新证书替代旧证书。

2.2 请求处理
--------

这个功能对于非Web应用非常有用。假设传统客户端使用hosts文件连接www.google.com:8000，而且我开启了代理。在这个文件中，www.google.com被重定向到127.0.0.1，我又创建了一个Burp监听在8000端口。现在我需要将全部流量重定向到原始状态（www.google.com:8000）。一种方法是在Redirect to host 和 Redirect to port中分别指定为www.google.com 和 8000。

![重定向流量](http://drops.javaweb.org/uploads/images/d129465b295d22fa4d068ba71738a933bcd12efa.jpg)

如果应用在同一端口连接不同的节点（例如，我们想代理流量通过80端口或者443端口），我们就不能重定向流量了吗？我们需要在Options > Connections > Hostname Resolution中设置，在下一个部分我将会解释。

如果我使用Burp将流量导入另一个代理工具，比如Fiddler或Charles，这也是有用的。

当我想要在Burp和应用之间把TLS剥离出来，或者我想把它加进去的时候Force use of SSL这个选项就有用了。有一个我时常参考的[文档实例](http://parsiya.net/blog/2014-06-25-piping-ssl/tls-traffic-from-soapui-to-burp/)。

### 2.2.1 Burp的透明代理

[更多信息](http://parsiya.net/blog/2015-10-19-proxying-hipchat-part-3-ssl-added-and-removed-here/#2-2-1-what-is-this-connect:6f44788618f3174bd28bf25248bb8608)请看上面链接中的2.2.1和2.2.2部分。老实说，跟着这整个系列来看Burp怎么作为代理来工作，你将会避免出现许多问题。

如果我们代理了一个代理敏感的客户端，它将会在实际启用TLS传输前发送连接请求给想要连接的目标。这会通知代理该往哪重定向流量。这是因为代理无法看到TLS层加密的TCP流量包。由于不知道怎么处理流量，CONNECT解决了这个问题。代理敏感的客户端会把代理视为浏览器。

非代理敏感的客户端就不在意他们的流量是否经过了代理。大多数这种程序都不使用代理设置或系统代理设置。程序依旧认为它在节点间直接发送流量，并没有被重定向到Burp。Burp可以将TLS层的数据代理并解密数据包，在包中通过头中的host字段找到原始节点。这就是Burp的透明代理。

可以在Proxy > Options开启。选择代理监听，点击编辑，在Request Handling 中选择Support invisible proxying (enable only if needed)。

![透明代理](http://drops.javaweb.org/uploads/images/7ca6f2ecb2e287e542bc516be2ef8d1d2b53d004.jpg)

我经常用它来捕获应用和Burp之间的本地流量，判断应用是否发送连接请求。另一种方法是都尝试一下，看哪个好使。

2.3 证书
------

我们可以配置Burp的中间人证书。

*   Use a self-signed certificate意味着Burp只能使用单一证书
*   Geneate CA-signed per-host certificates是最常用的，Burp将会针对不同的host生成不同的证书。证书的CN和域名相同。
*   Generate a CA-signed certificate with a specific hostname这里我们可以指定证书的CN。当一个应用通过检查CN来锁定时就有用了，但是这与终端节点常用的通配符不太一样。
*   Use a custom certificate (PKCS#12) 如果我们有确定的证书需要使用，我们可以在这里修改。当证书检查机制并不只是检查CN，我们就需要手动输入证书。

0x03 额外提示：请在提供充足的内存运行Brup
=====

我从来没有在内存不够的电脑上运行Brup。但是我通常会在一天结束的时候保存工作状态，我不会大量使用Python/Ruby的扩展，而是YMMV。 通过命令行来给Burp分配2GB的内存：

`java -jar -Xmx2048m /burp_directory/burpsuite_whatever.jar`

0x04 Scope
=====

理想情况下，你想添加应用程序的终端节点到Scope，这会帮你过滤掉其他无用的噪声。

把终端节点添加到Scope中很容易，右键单击一个请求，然后选择Add to scope即可。然后你就可以在Target > Scope中看到你刚刚添加的请求。

只有你指定的那个URL才能进入Scope中，这很方便。例如，我选择使用GET请求得到Google的图标，并且添加其到Scope中。Google.com剩下的数据包将不会包含在Scope中，你必须手动添加。

![添加Google Logo](http://drops.javaweb.org/uploads/images/f29b10db62bde49875303d986afe0551533f0ede.jpg)

另一种方法是复制URL粘贴过来。如果我们右键点击任何位置的任何请求，我们都能通过点击右键菜单中的Copy URL来复制这个请求的URL地址。我们可以在Scope中点击Paste URL来把这个请求置入Scope。这个按钮在其他位置出现也具有类似的功能。

![粘贴URL](http://drops.javaweb.org/uploads/images/4f5c2ce47c4e6c5c5822ff2f39d7b2eca5687dd9.jpg)

我们通常不止是需要一个请求。通常是整个节点，或者确定的整个目录。幸运的是，在指定Scope时，我们可以使用正则表达式。我通常使用上述方法之一将URL添加进Scope中，然后通过编辑按钮来对其修改。

Scope中有四个选项：

*   Protocol：可以选择Any、HTTP或HTTPS，我通常选择Any
*   Host of IP range:：支持正则表达式，例如*.google.com
*   Port：除非你为了寻找特定端口的流量，否则置为空即可。空意味着不会过滤掉任何端口的数据。这个选项也支持正则表达式。
*   File：这个选项也支持正则表达式。

如果我们想要添加Google和它的所有子域到Scope中，我们要添加这个Logo（或者任何Google的东西）到Scope中，然后编辑这个请求。

![添加Google.com的子域名](http://drops.javaweb.org/uploads/images/8d9103c47d5bbfd238e1788425173d27bb738c33.jpg)

0x05 HTTP历史
=====

在Proxy > HTTP History中可以看到Burp所有捕获的请求和响应。我几乎一半的时间都花在了这儿。我们可以使用过滤器来筛选数据包，过滤器的设置可以在Filter: Showing all items调整，你可以根据你不同的需求来调整你的设置。我一般都是清空这些状态重新开始。

![过滤设置](http://drops.javaweb.org/uploads/images/35d5752547f1f78838ce28a098062b058b76db72.jpg)

最有效的过滤是Show Only in-scope items，这将会把不在Scope里的数据包都清除掉。

![过滤](http://drops.javaweb.org/uploads/images/201baeddd6b009a63875d19436128c78fee3a7c2.jpg)

正如你在上图中看到的，过滤器具有很多设置选项。大部分的设置都非常简单易用，我要讲解的是那些我在非Web应用中常用的设置。

*   Filter by MIME type会一直保持所有的活动状态直到你确认自己没有丢失任何请求。MIME-type并不总能正确的反映请求。Other binary被用来查看大多数二进制或独特的payload（默认是不活跃的）。
*   在Filter by file extension中，我只使用Hide功能，通常会为那些我不喜欢的东西添加一些额外的扩展(比如字体)
*   当该应用程序在不同的端口进行通信，并且不支持代理设置时，Filter by listener就非常有用了。在这种情况下，我们必须重定向节点到localhost（例如使用host文件），还要为每个端口创建不同的代理监听。使用这个功能后我们可以查看特定监听器上的流量。

0x06 扫描(只在Pro版本中可用)
=====

Burp有一个特别好的扫描模块，它虽然不像IBM的Appscan那样优秀的覆盖面和准确度，但是它的优势体现在非Web应用测试。

*   Burp设置更为简单快捷，我可以登录、扫描单个请求。而在Appscan中，我必须立刻对整个应用程序进行测试（配置扫描、记录登录、手动探索、自动探索、完全扫描等）。而且Appscan还不能配置一些特殊的登录设置，比如随机令牌。
*   Burp便宜的多，一年授权300美元，而Appscan一年授权需要20000美元。

扫描结果在Issue activity选项卡中显示，在Target > Site map纵览全局结果则更为直观，扫描请求都在Scan queue中。

6.1 实时扫描
--------

Burp有两种扫描模式，主动扫描和被动扫描。两种扫描状态可以同时开启。 在被动扫描中，只关心请求和响应的数据包，仅仅通过规则进行匹配扫描，不发送任何请求。在主动扫描中，可以产生有效的测试载荷发送到服务器，同时分析其请求和响应。

在此可以对其进行配置：

*   永远不用让Live Active Scanning保持On的状态。永远不要这样做，你有可能会对应用造成破坏，或者应用被锁定。始终扫描每个单独的请求。
*   设置Live Passive Scanning为Scan everything可以增强扫描效果，但是会在结果中引入噪声。如果你设置了Scope，你可以将扫描范围限定在suite scope中。你还可以选择Use custom scope来自己根据需要进行定制Scope。

![实时扫描设置](http://drops.javaweb.org/uploads/images/d1d0d65af33a519510a6d1eb21977fc14d5bd7de.jpg)

6.2 设置选项
--------

在这里你可以配置这些扫描设置。尽管我不使用实时主动扫描，这些选项也可以用来配置对单个请求的扫描。

*   Attack Insertion Points:你可以自己选择注入点，根据你的需要添加或移除。减少注入点就等于提高扫描速度。
*   Active Scanning Engine:对某些服务器要调节扫描的最大流量，当你每分钟发出超过设置值的请求时，服务器有可能会将你的IP封掉。
*   Active Scanning Areas:正如你所看到的，这些设置是面向Web应用的。我取消了所有设置，只添加那些我认为相关的设置。这样在扫描期间可以节约很多时间。
*   Passive Scanning Areas:你可以取消一切设置，只添加那些减少噪声的设置，但是说实话，连我都懒得去改动这些设置。
*   Static Code Analysis:开启对JavaScript代码的静态分析，对于非Web应用测试，你大可将其关闭。

0x07 入侵（Free版本有速度限制）
=====

入侵模块是Burp的半自动扫描部分。可以右键单击任何一个请求发送到入侵模块，在入侵模块中你可以指定注入点，之后可以使用内部扫描器来进行扫描，或者使用你自己的载荷。

7.1 位置
------

发送载荷到入侵模块后，在Intruder > Positions标签页中看得到注入点。我通常先Clear掉所有设置，然后使用Add来对自己设置的注入点进行标记。

![入侵模块](http://drops.javaweb.org/uploads/images/5ba31d9e3880c1420bf73b565432ef90c9fede8d.jpg)

正如你所见，我选择之前用过的Google的Logo再次发送到入侵模块。然后清除掉所有的预定义的注入点和已经添加的文件名。现在我们可以右键点击并选择Actively scan defined insertion points来将其发送到扫描模块，不仅扫描注入点还可以使用Start Attack来用我自己设计的载荷进行注入。因为我没有选择任何载荷，第二项是不会起效果的。

7.2 Payloads
------------

现在我们可以设置我们自己的载荷来配置入侵模块进行攻击。切换到Payloads选项卡。

### 7.2.1 载荷设置与选项

在这，我们可以使用不同的载荷。Burp可以满足你在特殊情况下进行一些复杂的负载设置。例如，Recursive grep可以让你在先前载荷的响应中得到每个载荷。还有Case modification、Character substitution、Dates和Numbers等等。

Simple list允许你对所有的注入点都使用自己的载荷。在有载荷的源文件中复制到Payload Options中。你可以直接选择使用这个文件。

Runtime File通过点击Load按钮来加载一个文件。你也可以使用Burp自带的负载列表，但似乎只有Pro版本才有。

![内置简单列表](http://drops.javaweb.org/uploads/images/92e6725d575f898c743efa8e7430704d509eb9de.jpg)

Custom iterator（自定义迭代器）可以让Burp生成更加复杂的载荷。如果我想要模拟两个自己的十六进制（四个字符），我可以在一个位置上设置0-9和a-f在其他位置上也类似的设置。Burp允许你设置八个位置。如果你想添加用户名，可以选择五个位置，添加自己的用户名列表。Burp也支持位置间使用分隔符。

![自定义迭代器](http://drops.javaweb.org/uploads/images/a65b533bf586213398a0d86f5ac41f8144c7e728.jpg)

一个流行的载荷列表是[FuzzDB](https://github.com/fuzzdb-project/fuzzdb)。需要注意的是，这些载荷基本都会触发反病毒软件的报警。

### 7.2.2 载荷处理

在进行测试之前，你可以对载荷进行转换处理。例如，你可以将载荷进行base64编码再发送，或者发送其hash。

![载荷处理](http://drops.javaweb.org/uploads/images/f1b2f94fd9d90e220256ca65bc3a9114eeabe75e.jpg)

7.3 设置
------

我们可以在Request Engine限制入侵模块的速度，就像扫描模块那样，或者减慢其速度。Payload Encoding可以在Burp中对特殊字符进行URL编码。

### 7.3.1 正则匹配

我们可以查找带有特定字符串的响应。如果我在进行SQL注入测试，我就可以寻找带有SQL错误信息的那些响应。对于XSS，我通常使用9999进行测试，之后只要在响应中匹配9999即可。

FuzzDB有自己的正则表达式来分析响应，这个[页面](https://github.com/fuzzdb-project/fuzzdb/wiki/regexerrors)展示了如何使用它们。

0x08 重放、解码、比对
=====

虽然这些模块没有很多功能，但是也相当有用。

8.1 重放
------

重放模块是为了手动测试而设计的，扫描模块是自动化，入侵模块是半自动化。

将请求发送到重放模块和之前类似。我们可以修改请求，发送请求然后观察响应。

我选择Google的Logo的GET请求，发送到重放模块然后发送它。我们可以对其进行修改，然后查看其404响应和一些无效文件。用Ctrl+Z可以撤销这次更改。

![重放模块](http://drops.javaweb.org/uploads/images/5196bfff510510e30efd775b928bcc9db3d5ea73.jpg)

你可以将修改过的数据包发送到扫描模块或者入侵模块进行进一步扫描。

8.2 解码
------

允许使用不同的格式进行编码或解码。也支持创建其哈希。双击任何一个参数右键发送至解码模块。你也可以使用Ctrl+C复制其到解码模块。

![解码模块](http://drops.javaweb.org/uploads/images/90f52830e81ffb5f93013e4c108653ab0cd20192.jpg)

8.3 比对
------

比较模块可以对两个载荷、两个HTTP请求或响应进行比对。可以在字节水平（通常是二进制对象）进行比对，也可以在字的水平进行比对（通常是文本）。

0x09 配置选项
=====

这个模块是用来配置Burp进行更复杂的工作。

9.1 连接
------

### 9.1.1 平台认证

如果应用程序需要特殊形式的身份验证，如NTLM或Basic，你都可以在这里进行配置。如果你需要在浏览器中进行身份验证，那么你可以不需要进行修改。你将会在Burp中看到消息头的部分，意味着你不需要每次登录应用时都输入它们。如果你需要使用某种平台进行身份验证，Burp对传统客户端都是有帮助的。

有时你的工具并不支持对其进行设置，我在使用Appscan时就存在这个问题，尽管Appscan支持平台身份验证。

你可以重定向你其他工具的流量到Burp，让Burp承担起这个责任。我不是暗示它们会生成大量的流量，但是形势所迫，你必须使用Appscan。如果验证失败，错误消息将显示在Alerts选项卡中。

![平台认证](http://drops.javaweb.org/uploads/images/243c60d7d034675542d2c41083fd69b8f3b01304.jpg)

开启Prompt for credentials on platform authentication failure，保证在认证失败时可以迅速传递给浏览器。

### 9.1.2 上游代理服务器—Socks代理

我已经讲了很多关于Burp如何作为一个代理链中的一部分。我们要在这里对经过Burp转发的请求进行设置。这对联合代理服务器环境下的Burp使用是非常有帮助的。通常情况下，这些代理服务器是自动配置的。这些设置可以访问Tools (menu) > Internet Options (menu item)> Connections (tab) > LAN settings (button)来更改IE的代理设置。通常在Use automatic configuration script中能够对proxy auto-config和pac文件进行配置。检索PAC文件并且在文本编辑器中查看，应该能看到代理地址:端口格式的配置。

![上游代理服务器](http://drops.javaweb.org/uploads/images/52854d63e52aff843fffc86cb51ce18252d6ad20.jpg)

使用SOCKS代理是类似的。根据细则，这将会覆盖之前的代理设置。就我个人而言，我从来没有对Burp的Socks代理进行配置。

### 9.1.3 超时设置

对老服务器进行测试，应该增加超时设置，如果Burp是代理链中的一部分，就要增加超时的设定，以修正延误。

![超时设置](http://drops.javaweb.org/uploads/images/8356944389382f76ec97496f2486632d4f6e6853.jpg)

### 9.1.4 主机解析

我在之前简要讨论过。如果应用使用多个端口还不支持代理服务器设置。我们就要将应用程序的流量重定向到Burp。现在Burp需要知道这些流量将要被转发到哪，否则它会陷入本地回环的无限循环中。

因为Request Handling只支持一个节点，我们不能使用它。正相反，我们会将其置空，并添加节点及其关联的IP地址在Hostname Resolution中。例如，server.com和10.11.12.13。如果节点经过负载均衡、CDN或者像亚马逊的S3共享主机，就会更加复杂。在这种情况下，不经过代理，运行一个Wireshark或者Netmon来捕获应用程序的流量。发现HTTP请求发送目的地的IP地址。使用这个IP在host头字段中发挥作用。有许多方法能做到这一点，但是我觉得这个方法是最简单的。

### 9.1.5 超出域的请求

我们可以指定Burp把不属于域内的请求全部丢弃。如果我们设置了正确的域范围，就可以降低流量噪声，这会对我们有利。如果应用程序连接了除了我们正在测试的节点外的其他节点，这样就增加的域的范围，它可能会停止工作。如果你是一个没有使用过Burp的人，又不想让这个功能起作用，你可以在此指定。除此之外，我没觉得它有多么有用，我从未使用过此功能。

9.2 HTTP
--------

Redirections和Status 100 Responses都很简单，我就跳过了这个部分

### 9.2.1 流响应

这是一个被低估的功能，特别是对非Web应用测试来说。当我们要使用这个功能时，先让我们看看代理是如何工作的。我曾经写过一篇[文章](http://parsiya.net/blog/2015-10-19-proxying-hipchat-part-3-ssl-added-and-removed-here/#2-how-does-a-proxy-work:6f44788618f3174bd28bf25248bb8608)解释这件事儿。

总之，下面是流程：

![GET流程](http://drops.javaweb.org/uploads/images/71ee241499f8de3b18715aa6cc4feed2b70f586b.jpg)

1.  Hipchat creates a TCP connection to Burp.
2.  Hipchat sends the GET request to Burp.
3.  Burp creates a TCP connection to Server.
4.  Burp sends the GET request to Server.
5.  Server send the web page to Burp.
6.  Burp closes the TCP connection to Server.
7.  Burp sends the web page to Hipchat.
8.  Burp closes the TCP connection to Hipchat.

需要注意的是，这个图和HTTPS请求不同。

现在假设`http://downloads.hipchat.com/blog_info.html`是个大文件（比如是100MB的更新），应用程序需要请求下载。该应用程序将此文件作为流，根据下载完成的数据量显示一个进度条。

如果我们代理这个请求，Burp会执行步骤四。直到下载完成，Burp才会将此数据发送到应用程序中。这意味着应用程序会等待此文件，可能就放弃等待了，并重新发送请求或者返回超时或冻结。如果我们添加`http://downloads.hipchat.com/blog_info.html`到Streaming Responses部分，Burp就会在接收到数据时立即把响应传递给应用程序。

9.3 SSL
-------

Server SSL Certificates只显示从服务器中检索的证书列表。使用命令行工具像Openssl就可以很容易的得到证书，但是我猜你也想在这里查看证书信息。

### 9.3.1 SSL协商

无论如何，安装[其](https://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)都是个好主意，但是你也可以在Burp中使用无限长度的密码。要确保你运行的是最新版的Java运行环境（JRE）。

除了Disable Java SNI extension启用所有选项，只有当你需要SNI的时候再启用。Allow unsafe renegotiation看起来有点吓人，但是当使用客户端证书的时候，这非常有用。

在代理期间，如果你不确信当前TLS设置是否工作，你需要时刻留意Alerts标签页。如果TLS我收失败或者Burp和服务器无法完成TLS握手，就会有错误提示。对于错误请看这个[页面](http://parsiya.net/blog/2016-03-27-burp-tips-and-tricks-for-non-webapp-testing---part-1-interception-and-proxy-listeners/#1-4-ssl-pass-through:dd27d7b63bfec82bf20dab0a96096152)。

### 9.3.2 客户端SSL证书

如果应用程序需要客户端证书，我们也可以非常容易地添加。我们可以选择目的host和Burp对应这个host使用的证书。

9.4 会话
------

这个选项卡提供给我们许多自动化的东西。你可以创建宏。你可以进行一些操作然后把它们记录下来录制为宏。之后在中， 你可以在确定的域内或对特定的请求来选择要运行的宏。例如，你可以创建用于登录的宏，这样Burp可以在你发送请求前就登录。还可以创建一个带有特定值的参数，并将其添加到每个请求，或者在一定范围内的请求，或者自动改变参数的值。

9.5 界面
------

你可以更改字体、编码和其他一些可以更改的东西。不加赘述。

9.6 杂项
------

在这里大多数项目不需要任何解释。如果你有Pro版，我建议你开启Automatic Backup。我通常将其设置为每小时存档并且退出时存档。因为，我希望可以每天保存状态的备份，我也开启了Include in-scope items only，这大大减少了保存数据的体积。 如果你需要在测试期结束时回头检查一些东西，或者你的账户被锁定时，你可以回档。 计划任务会允许你做一些计划。例如你可以设置扫描模块在何时启动。如果你不想或者不能每天扫描，那就让它在特定的时间进行扫描吧。不幸的是，作为计划任务不允许运行宏，并且可选项也非常有限

![计划任务](http://drops.javaweb.org/uploads/images/0bdd144c9f33482b54a97f3d75f7de6998264181.jpg)

### 9.6.1 Burp默认服务器

这是一项新功能。默认设置在Use the default collaborator server。这是我非常不喜欢的一个功能，每次我新安装完Burp总要去修改这个设置。我不太清楚它会把什么信息传到服务器中，所以我宁愿别发送客户端信息到默认的服务器。你也可以选择运行，详细的可以阅读[文档](https://portswigger.net/burp/help/collaborator.html)

0x0A 报警
=====

报警标签页也是重要的。特别是在TLS连接问题或是超时。如果这个标签亮起时，要注意这个标签。

0x0B 扩展模块
=====

Burp支持扩展，扩展可以用Java、Python或者Ruby编写。不幸的是，没有很多非Java开发的扩展。个人来讲，我更喜欢Python。我可以从读其他人编写的扩展中学到更多。

11.1 扩展
-------

这个选项卡会显示当前被加载的扩展。它还能显示这些扩展的输出和错误。在扩展加载之后，就要关注它的错误页面运行中是否存在错误。添加扩展非常简单，单击添加，然后选择类型和扩展文件的路径。我通常把这些都放在Burp目录的一个子目录内。

11.2 BApp商店
-----------

在Burp的app store中安装插件是一件轻而易举的事情。只需要切换到该选项卡，选择扩展名然后单击安装。如果该扩展是用Python编写的，你必须安装Jython才能运行。如果不方便安装，Burp很方便的提供了下载Jython按钮。

![Jython](http://drops.javaweb.org/uploads/images/fb2c24ab7b855efe61696857535bedcd19f2112f.jpg)

单击下载按钮打开此页，下载最新的Standalone Jar。我通常把它放在和Burp相同的目录中。之后在Extender > Options中，可以选择Python Environment > Location of the Jython standalone JAR file其作为你的运行环境。

![Jython路径](http://drops.javaweb.org/uploads/images/62a679336fd1e12d9764fd2819ab919e8b4f23f6.jpg)

现在就可以点击安装按钮进行安装了。

11.3 API
--------

APIs选项卡中有API文档。Burp扩展可以使用这些API来与Burp进行交互。正如你在那些为Java扩展编写的文档里看到的。

11.4 设置
-------

正如之前所见，我们可以为Jython和JRuby设置路径。我们也可以为Java和Python编写的扩展指定目录。在Burp启动时，这些扩展所在的目录会自动加载。

0x0C 实际案例
=====

使用到的应用程序

Cygwin + IBM Appscan Standard + Charles Proxy + Fiddler + SoapUI

你不需要使用Burp的Pro版，IBM Appscan 也用的是评估版。所有的应用都是免费使用的。

12.1 开始
-------

一般情况下，应用程序有自己的代理设置或者使用IE的代理设置，我们都能够使用Burp来进行代理。

在开始之前，要[确保](http://parsiya.net/blog/2016-02-21-installing-burp-certificate-authority-in-windows-certificate-store/)你在操作系统证书存储区安装了Burp的根证书。

Burp的默认代理监听在127.0.0.1:8080

12.2 Cygwin
-----------

Cygwin是一个Windows下的类*nix命令行。使用Cygwin可以很方便的运行很多程序，比如git或Curl。因为在一个API测试中，我需要使用一系列Curl命令。为了方便，我通过Cygwin来执行。重定向到Burp中，并且在测试中将使用到重放模块。

为了在Cygwin中安装Curl，我们需要运行Cygwin的安装文件，并且在可用的包列表中选择curl。这和包管理器的功能是相似的，并且支持搜索功能。所以安装Curl是非常容易的。安装文件会自动下载其依赖。

我要用再次使用GET请求再一次来取回Google的Logo。使用Burp模拟一些Curl命令来完成这个最简单的任务。重定向浏览器的流量到Burp，并访问Google首页。确保你清除了浏览器缓存，否则将不会发送请求，你也看不到请求了。选择Logo图片的那个请求，右键点击Copy as curl command。

![拷贝Curl命令](http://drops.javaweb.org/uploads/images/7585e2e8c1dde35b4a53114144f93dd0e52d11e9.jpg)

结果应该与下面类似：

`curl -i -s -k -X 'GET' \ -H 'Referer: https://www.google.com/' -H 'User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko' -H 'DNT: 1' \ -b 'NID=78=S4kjzm5kwP82gAN8xazSJCiG6UWZhRNEGEE_a3hHZ2OMcy5bPX1CjZisClbvBgPUodlcpywR6WyhVSRUykloTI3ay7jSy9fpgTG2tKV2s8eojpQmL_F5sYKyHP1exm8iwp0F_FEnnE_DaQ' \ 'https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png'`我们不需要关心这么多项，我们可以忽略Referer、Cookie、User-Agent和其他一些没用的东西。

`curl -i -s -k -X 'GET' 'https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png'`

### 12.2.1 -k/安全开关

-k意味着Curl将不会验证证书的安全性。在使用Burp时，有时候会非常必要，因为Burp自签发的证书有可能失效。如果你想使用其他Curl命令，一定要记得加上-k来保证不出现任何问题。

### 12.2.2 设置Burp为Cygwin的代理

这很简单，我们只需要执行以下命令：

`export http_proxy=http://127.0.0.1:8080 export https_proxy=http://127.0.0.1:8080`

![重定向](http://drops.javaweb.org/uploads/images/a28f54d04639e65528cd7b3e6bd684caefaa7af3.jpg)

12.3 IBM Appscan Standard
-------------------------

众所周知，这是一个Web应用扫描软件。有时候我会将其重定向到Burp上，为了练习，我们将使用评估版，在撰写本文时，版本是9.0.3。我们必须注册一个免费的IBM ID才能获得软件，我相信这对你来说并不难。

评估版只能扫描IBM的示例网站`http://demo.testfire.net/`，不过这就足够了。

在Appscan中，我们可以在Scan Configuration > Connection > Communication and Proxy设置代理。这和Firefox的代理设置类似，我们也可以使用IE的代理，或是为Appscan指定一个特定的代理。我们有两种方法，第一种是通过IE代理，第二种是直接设置代理。

![Appscan代理设置](http://drops.javaweb.org/uploads/images/525a205643e11e55f689cc71656d58bb1edce420.jpg)

现在，我们可以重定向Appscan的流量到Burp了，也可以进行一些手动探索了。试用版软件可以让我们这样做，但是不要把它添加进扫描中。如果我们有Appscan的授权版本，我们可以运行一个扫描。在Burp中可以看得到扫描的流量经过。

或者，你也有可能想把其他的流量导入Appscan中。例如，使用额外的浏览器或者其他工具，你可以在Tools (menu) > Options > Recording Proxy (tab) > Appscan proxy port中设置导入端口。你也可以通过选择，来指定你自己设置的端口。在手动探索时Appscan只是起到监听功能。

![代理设置](http://drops.javaweb.org/uploads/images/d6ed00d1b8619420163dd838813c8e7ff5aec2d3.jpg)

你可以安装导入Appscan的根证书了。

12.4 Charles Proxy
------------------

有时候，你需要使用两个或两个以上的代理来组成代理链。我现在将要给你介绍最流行的HTTP代理。

我们先下载并安装一个免费试用的[版本](https://www.charlesproxy.com/download/latest-release/)，在截稿时，版本是3.11.4。出于演示目的，我会使用IE和普通的Google主页。事实上流量可以来自我们关心的任何位置，只要代理成功，其余东西都是相同的。

要确保你安装了Charles的根证书，你可以在Help (menu) > SSL Proxying设置。

### 12.4.1 IE -> Burp -> Charles

首先你要在Proxy (menu) > Windows Proxy关闭Charles的自动代理设置。如果系统代理是开启的，就会在下拉菜单中有一个小小的标记。在Proxy (menu) > SSL Proxying Settings (sub-menu) > SSL Proxying (tab)中确保Enable SSL Proxying是被选中的。点击添加，在HOST和PORT上填入*。在Proxy (menu) > Proxy Setting (sub-menu) > Proxies (tab)里我们可以查看HTTP代理端口，默认是8888端口，我们也可以指定其他端口。

![Charles SSL代理](http://drops.javaweb.org/uploads/images/17451dafa89da5299f16c756e7c4e6e5318149cf.jpg)

试用版本会每隔三十分钟重新启动，就会再次开启自动代理。确保每次启动后你都要禁用它。你也可以使用Firefox来代替IE代理设置。

在Burp中，可以在Options > Upstream Proxy Servers添加上行代理服务器为127.0.0.1:8888

![配置Charles作Burp的上行代理](http://drops.javaweb.org/uploads/images/ef9a0ed823a0ed19cf6990282c0343feb06cba01.jpg)

现在可以在IE中打开Google的主页了，可以看到流量经过了Burp和Charles。

![流量展示](http://drops.javaweb.org/uploads/images/02b714733ee743ed3a84cbc6dc459e30e488f854.jpg)

### 12.4.2 IE -> Charles -> Burp

我们只需要将IE的代理设置改为Charles，即127.0.0.1:8888。幸运的是就算Charles每隔三十分钟重启，但是它不会重置设置。

Charles要在Proxy (menu) > External Proxy Settings (sub-menu)确保开启了Use external proxy servers。要将Web Proxy (HTTP)和Secure Web Proxy (HTTPS)都输入Burp的代理监听（127.0.0.1:8080）

![外部代理设置](http://drops.javaweb.org/uploads/images/9744ed0cc3309881c416ec9bd3ac654204fd4c1e.jpg)

设置完成，可以打开主页了。

![流量展示](http://drops.javaweb.org/uploads/images/2927a7ca40851fd45845b065dd6dbd8fcbe3428a.jpg)

12.5 Fiddler
------------

Fiddler的好处在于其可以添加脚本，我们使用的是4.6.2.2这个版本。

### 12.5.1 IE -> Fiddler -> Burp

首先，我们要在Tools (menu) > Fiddler Options (sub-menu) > HTTPS (tab) > Decrypt HTTPS traffic设置Fiddler可以捕获HTTPS流量。这将会在Windows证书存储区安装Fiddler的根证书。这将会让Fiddler捕获并且解密全部经过代理的流量。还要确保Ignore server certificate errors (unsafe)，因为Burp可能是不被承认的。

![Fiddler HTTPS选项](http://drops.javaweb.org/uploads/images/016b2555d6bbf61969eaca23e10fbce0d2989290.jpg)

切换到Gateway标签页，选择Manual Proxy Configuration并且在上面的输入框中输入http=127.0.0.1:8080;https=127.0.0.1:8080，这将会将Fiddler的流量导入到Burp。

![重定向流量](http://drops.javaweb.org/uploads/images/135fe9d8a8d82243262d110594a88a040de6a193.jpg)

现在设置完成，可以访问首页了。

![流量展示](http://drops.javaweb.org/uploads/images/e02d1a783e5e7a51b8ebcce7dda8c412c7886ac8.jpg)

你也可以在Tools > Fiddler Options > Connections (tab)手动更改Fiddler的代理设置。

![Fiddler设置](http://drops.javaweb.org/uploads/images/184db3861e3e65c46bc95145b8018c86a8f716ad.jpg)

### 12.5.2 IE -> Burp -> Fiddler

首先要禁用Fiddler的自动代理，并且把Gateway中的所选项删除。如果使用IE代理设置，记得选择No Proxy，否则又会回到Burp。之后我们设置Burp是IE的代理，Fiddler是Burp的上游代理。

12.6 SoapUI
-----------

这个部分我已讲过，请[参见](http://parsiya.net/blog/2014-06-25-piping-ssl/tls-traffic-from-soapui-to-burp/)。

作者的案例要写在第五部分，不知道会连载几期

*   Part Ⅰ[原文地址](http://parsiya.net/blog/2016-03-27-burp-tips-and-tricks-for-non-webapp-testing---part-1-interception-and-proxy-listeners/)
*   Part Ⅱ[原文地址](http://parsiya.net/blog/2016-03-29-burp-tips-and-tricks-for-non-webapp-testing---part-2-history-intruder-scanner-and-more/)
*   Part Ⅲ[原文地址](http://parsiya.net/blog/2016-04-02-burp-tips-and-tricks-for-non-webapp-testing---part-3-options-and-extender/)
*   Part Ⅳ[原文地址](http://parsiya.net/blog/2016-04-07-burp-tips-and-tricks-for-non-webapp-testing---part-4-burp-in-proxy-chains/)
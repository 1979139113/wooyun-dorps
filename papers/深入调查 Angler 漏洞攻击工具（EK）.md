# 深入调查 Angler 漏洞攻击工具（EK）

> [https://blogs.sophos.com/2015/07/21/a-closer-look-at-the-angler-exploit-kit/](https://blogs.sophos.com/2015/07/21/a-closer-look-at-the-angler-exploit-kit/)

0x00 前言
=====

在过去的几年中，犯罪分子广泛地采用EK（漏洞攻击工具）来实施木马感染活动。EK工具通常会出现在名叫[Drive-by](https://blogs.sophos.com/2014/03/26/how-malware-works-anatomy-of-a-drive-by-download-web-attack-infographic/)的攻击阶段，”drive-by”能够悄悄地把用户浏览器定向到一个恶意网站，而在这个网站上就会放置着某个EK。

然后，EK就会利用安全漏洞，让用户感染木马。整个过程可以是完全隐藏的，并且不需要用户进行操作。

在本文中，我们研究的就是臭名昭著的Angler EK。

0x01 介绍
=====

Angler最早出现在2013年末，自此以后，Angler在犯罪市场中的受欢迎程度与日剧增。为了绕过安全产品的检测，Angler使用了多种组件(HTML, JavaScript, Flash, Silverlight, Java等)，以及这些组件的不同变种。另外值得一提的是Angler非常活跃。比如，在2015年5月，我们每天都能发现上千个新出现的Angler陷阱网页，也就是所谓的“Landing page”。

从每周检测量来看，Angler从2014年中期开始出现，在2014年末突然爆发，接着稍有缓和，然后自2015年起，活动数量又开始攀升：

![p1](http://drops.javaweb.org/uploads/images/09448fc82e3c347b6eb8963c9a4f20b9a106db8b.jpg)Figure 1-根据每周检测量画出的Angler增长趋势

由于[Blackhole EK](https://nakedsecurity.sophos.com/2013/10/08/assessing-the-impact-of-the-blackhole-arrests/)在2013年10月停止了活动，相关的攻击者也被逮捕，所以其他的EK得以发展，并分割了市场份额，但是占主导地位的还是Angler EK。为了显示Angler的受欢迎程度，我们分析了在3个不同时期中，各个EK的活动情况（2014年9月、2015年2月、2015年5月）：

![p2](http://drops.javaweb.org/uploads/images/aabadaab5c23818710f50dd74599d07ebc97b781.jpg)图2-在2014年9月，2015年2月和2015年5月，根据每周检测数据衡量的EK活动分布

0x02 流量控制
=====

很明显，任何EK只有利用了目标计算机上的漏洞并安装上木马，才能算是成功。但是，EK还需要大量的输入流量，才能掌握潜在受害者信息。对于大多数drive-by下载而言，这一点是通过入侵合法网站并注入恶意HTML或JavaScript来实现的。受害网站的级别和受欢迎程度越高，EK能获取到的流量就会越多。

我们已经发现了好几种能把用户的web流量发送给Angler的技术 。其中有很多都利用了简单的[IFRAME](https://en.wikipedia.org/wiki/HTML_element)(嵌入式 HTML框架)注入，注入的一般是HTML或Javascript，没有多少意思。但是，Angler使用的某些重定向方法更特殊，值得在这里一提。

**2.1 HTTP POST重定向**

在前面，我提到过一般来说用户是看不到重定向过程的。实际上，情况也不一定总是这样，比如，就从2014年5月的第一个例子来看，我们发现大量的合法网站中既注入了JavaScript也注入了HTML，如下。不同于常规注入，在这里的某些JavaScript中还增加了FORM和DIV元素：

![p3](http://drops.javaweb.org/uploads/images/aacd86b10ae04f8d2cf00dcaebc27cc793ebe5e6.jpg)图3-注入的JavaScript和HTML（检测为Troj/JSRedir-OA）

在加载页面时，用户会看到一个对话框，提示用户点击“Yes”或“Cancel”。根据注入的JavaScript来看，无论点哪个，Go() 函数都会运行，移除这个弹窗，添加一个IFRAME占位符，然后提交表单：

![p4](http://drops.javaweb.org/uploads/images/593ba4d2364062ce420fd41c0dfe07b4054cb92d.jpg)图4-当用户的浏览器注入Troj/JSRedir-OA时显示的弹窗

表单发送的数据包括3个值（经过编码），可能是为了协助犯罪分子管理这一重定向机制：

*   IP地址
*   User-Agent 字符串
*   URL

接着，响应POST表单的是HTML和JavaScript（加载到IFRAME占位符中），用于重定向用户：

![p5](http://drops.javaweb.org/uploads/images/aa5081ac59b3b6085af9775a8af0fd0b414b2f07.jpg)图5- HTTP POST请求的响应中包含用于进行重定向的JavaScript

几个月后，我们发现了相似的技术，但是这次是一个Flash组件。被攻破的网站经过篡改，会包含有HTML，这个HTML会加载来自另一个被攻破网站的恶意Flash文件。

然后，这个Flash文件中的ActionScript 会获取各种参数，并发出 HTTP POST 请求。

![p6](http://drops.javaweb.org/uploads/images/1e2317d0efa8bfb087a86a5372a97c880e3f594c.jpg)图6-在Flash版 Troj/JSRedir-OA中使用的ActionScript

HTTP POST请求的响应和以前一样（图5），返回的HTML和JavaScript会重定向用户。有趣的是，在2015年初，我们发现Nuclear EK也使用了后面这种重定向技术来发送流量。

**2.2 域名生成算法**

另外我们还观察到Angler 使用了域名生成算法（DGA）来进行重定向。在2014年，大量的合法网站都被注入了JavaScript，这个JavaScript中添加了一个脚本元素从远程网站上加载内容，这个远程网站的主机名是根据当前日期的哈希确定的。这种思路就是要每天都使用不同的域名，但是恶意脚本中不会有以后要使用的域名。这些DGA重定向曾经使用了.EU和.PW域名。

![p7](http://drops.javaweb.org/uploads/images/84d1528e8b93adc29e8cb6a5b685d9c8dc37d9f7.jpg)图7-使用DGA重定向的JavaScript中的代码段(Troj/JSRedir-OE)

这种重定向方法有缺点，一旦知道了这种算法，安全社区就能够预测特定日期时使用的目标主机名，并进行拦截，从而有效地阻止攻击行动。

**2.3 HTTP重定向**

本章中的最后一个例子利用了web重定向，在这种重定向中使用了HTTP响应代码302，这个代码通常是在合法网站上使用，用于转移其他网站上的访客。这里会涉及到内容注入，以及与重定向过程中的联系。在2015年4月，我们注意到用户在浏览_eHow_网站时会被重定向到Angler。通过分析其流量，我们确定了重定向过程，如图8。

ehowcdn.com 中的一个合法库中注入了JavaScript，似乎是从一个_Optimizely_服务器中加载内容（Optimizely提供了网站分析服务，通常是为了评估web广告的有效性）。但是，注意在optimizelys.com的主机名结尾有一个“s”，而合法的域名是optimizely.com。

![p8](http://drops.javaweb.org/uploads/images/68fbf506cd56c4972c395d88cb40a2b328207ec6.jpg)图8-近期伪装成Optimizely的重定向

1：eHow网站上的web网页，2：ehowcdn.com上的脚本会从optimizelys.com上加载脚本，3：用于添加恶意iframe脚本，4：302跳转到Angler登录页

在观察到这种情况的几天中，有[报告](https://twitter.com/kafeine/status/595708836314484737)称其他网站上也发现了相同的重定向(还是cdn3.optimizelys.com)-这次，最后的目标是Nuclear EK。

0x03 漏洞攻击工具
=====

**3.1 Landing page**

“Landing page”就是漏洞攻击工具代码的起点。通常，登录页(Landing page)上会使用HTML和JavaScript内容来识别访客的浏览器以及安装的插件，这样EK就能选择最有可能导致drive-by下载的攻击手段。

在Angler登录页上使用了多种混淆技术。除了增加分析难度，这些技术还能让犯罪分子更方便地根据每条请求制定不同的内容，从而绕过安全产品的检测。在过去的几年中，Angler一直会把自己的主脚本功能编码成数据字符串，储存到父HTML中。然后，当浏览器加载了登录页后，就会获取和解码这一内容。有很多恶意威胁都使用了这种[anti-emulation](https://www.sophos.com/en-us/why-sophos/our-people/technical-papers/malware-with-your-mocha.aspx)技术。

在解码了最外层的混淆后，就会暴露Angler使用的另一项技术-反沙盒检查。Angler利用了IE浏览器中的[XMLDOM](https://soroush.secproject.com/blog/2013/04/microsoft-xmldom-in-ie-can-divulge-information-of-local-drivenetwork-in-error-messages/)功能来判断本地系统上的文件信息。这样是为了检测系统上的安全工具和虚拟化产品。

![p9](http://drops.javaweb.org/uploads/images/42aece18b09ed61d544edbd40639495e986900ae.jpg)图9-Angler登录页上的代码段，这些代码会检测不同的安全工具和虚拟化产品

在反沙盒检测的下面是第二层混淆。某个函数会被调用，用于把几个长字符串转换成Unicode（双字节）数据，其中一些数据实际是随后添加到网页上的脚本内容。在大多数情况下是JavaScript，但是有时是VBScript内容（尤其针对的是[CVE-2013-2551](https://nakedsecurity.sophos.com/2013/05/14/may-patch-tuesday-critical-for-users-of-internet-explorer-and-web-based-services/)）：

![p10](http://drops.javaweb.org/uploads/images/0ca57c9532ec4b7113459ab2cf74de242bc640fb.jpg)图10-第二层混淆，隐藏shellcode数据和添加到页面上的额外脚本内容（在这里CVE-2013-2551是要针对的漏洞，所以JS和VBS内容都要添加）

在各个版本的登录页中，增加的脚本内容是为了针对特定的漏洞。但是，所有添加的脚本中都会有代码来：

*   动态构建 shellcode
*   从Angler登录页中的一个数组中获取数据（如下）

Angler登录页中包含有一个加密的字符串数组，在解码页面上，JavaScript函数会提供一个简单的代换密码来解码一个每个字符串。（有时候，编码字符串会储存到一个变量序列中，但是，更常用的是一个数组。）在木马中使用的秘钥通常是数组中的第一个字符串(uhBNwdr[0] 下)：

![p11](http://drops.javaweb.org/uploads/images/8162a33df2418853be9e0c4140d7c579d5a158d8.jpg)图11-在Angler登录页上使用的替换密码

这个数组的大小和内容会根据这版EK所针对的漏洞而变化。在数组中储存的数据包括：

*   托管EK的服务器名称
*   用于定位Silverlight内容的文件夹
*   用于定位Flash内容的文件夹
*   有效载荷 URLs.
*   Flash 有效载荷字符串 (编码的 shellcode 获URL数据)

必须要获取和解码这些数据。在这，你可以看到相关的有效载荷字符串都整合到了动态生成的shellcode中：

![p12](http://drops.javaweb.org/uploads/images/2bbe5486b7037b021e943b5092e3ec0fab5129c2.jpg)图12-用于创建unicode字符串的JavaScript函数，随后，当shellcode在利用CVE-2014-6332时，会解码这个unicode字符串

用于加载恶意Flash组件的代码很直接。假设已经通过了反沙盒检测，三个很有特点的(get*)函数会获取和解码登录页上的字符串。然后，这些字符串会用于创建HTML对象元素，然后这个对象会添加到文档中：

![p13](http://drops.javaweb.org/uploads/images/253cc05b93e08db3966eb36d667f99d5e664cea6.jpg)图13-用于加载恶意Flash内容的Angler登录页代码

**3.2 Flash内容**

经过多年的发展，Angler的Flash内容也发生了很大的变化。这些样本会使用不同的技术进行混淆，包括：

*   ActionScript 字符串混淆技术
*   Base64 编码
*   RC4 加密
*   Flash 混淆/保护工具 (比如，[DoSWF](http://www.doswf.org/)和[secureSWF](http://www.kindi.com/actionscript-obfuscator.php))

另外，Angler使用了内嵌的Flash对象。从登录页上加载的初始Flash并不是恶意的，而仅仅是作为一个loader通过一个内部Flash来投放漏洞。这个内部Flash可能是一个binaryData对象，或编码成ActionScript内部中的一个字符串。无论是哪种情况，数据会使用RC4加密。下面是一个1月份的例子：

![p14](http://drops.javaweb.org/uploads/images/e3d7bc4a44ee59e4d58f29f316515bf2d4ba1e6b.jpg)图14-最外部的Flash运载着使用了RC4和Base64编码的内部Flash

登录页上的数据会通过使用登录页HTML中的一个FlashVars参数，传递到Flash上-这是近期EK经常使用的一种技术。

通过loaderInfo对象的参数属性，就可以访问ActionScript 。很多EK都利用这种机制，以便：

*   在有效载荷URL数据中传递.
*   在shellcode中传递

这种灵活性能允许动态地修改shellcode，并且不需要重新编译Flash本身。

图15中是近期Angler Flash对象中使用的ActionScript代码段，用于从登录页上获取编码数据（“exec”变量）。另外，你可以使用某些函数中的控制流混淆来增加分析难度：

![p15](http://drops.javaweb.org/uploads/images/bac365b228d671725addbf3bd02f038ad800fcd2.jpg)图15-从登录页上获取编码数据(“exec” 变量)，然后通过ActionScript函数解码

在2015年初，又出现了一些Adobe Flash Player[0-day漏洞](http://malware.dontneedcoffee.com/2015/01/unpatched-vulnerability-0day-in-flash.html)（包括，CVE-2015-0310,[CVE-2015-0311](http://malware.dontneedcoffee.com/2015/01/cve-2015-0311-flash-up-to-1600287.html),[CVE-2015-0313](http://malware.dontneedcoffee.com/2015/02/cve-2015-0313-flash-up-to-1600296-and.html), CVE-2015-0315,[CVE-2015-0336](http://malware.dontneedcoffee.com/2015/03/cve-2015-0336-flash-up-to-1600305-and.html),[CVE-2015-0359](http://malware.dontneedcoffee.com/2015/04/cve-2015-0359-flash-up-to-1700134-and.html)），Angler很快就盯上了这些漏洞。

在过去的18个月中，越来越多的攻击活动利用了Flash漏洞，而不是Java，因为Oracle在[Java 7 update 51](http://java.com/en/download/faq/release7_changes.xml)中默认会阻止使用没有签名的浏览器。

**3.3 分析Shellcode**

如上，当浏览器加载登录页时，shellcode就会在脚本中动态生成。Shellcode的内容会根据目标漏洞而变化，但是，所有的编码方式和结构都是类似的。下面的分析介绍了针对CVE-2014-6332时使用的shellcode。

第一眼看上去，很明显shellcode数据（图10和图12中的shellcode_part1, shellcode_part2 和shellcode_part3 ）中并没有包含有效的代码。通过进一步的检查你就知道原因了-其他的数据是从VBScript组件中加载到了shellcode的开头。图12中的JavaScript build_shellcode()函数实际是从VBScript（图16）调用的。

![p16](http://drops.javaweb.org/uploads/images/19ac956d37304fccb09b1a0c90309c2e9f3bba0c.jpg)图16-VBScript的代码段，用于构建针对CVE-2014-6332 漏洞的shellcode

通过分析VBScript（图16中的buildshell1）提供的其他Unicode数据，证实了这些代码是可执行代码，并且在这些代码中实际包含了一个解密循环，用于解密shellcode其他部分中的字节，包括有效载荷URL和解密秘钥。这个解密循环会逆向图12中的encData() 函数，而有效载荷URL和有效载荷秘钥就是在这里加密的。

![p17](http://drops.javaweb.org/uploads/images/9b12478f5df32943b71aeb9b28cfec4aded445a5.jpg)图17-shellcode的代码段，例证了VBScript提供的解密循环

我们又检查了另一个登录页样本，这次针对的是CVE-2013-2551，这个样本使用了同样的技术，但是这次，解密循环添加到了JavaScript中：

![p18](http://drops.javaweb.org/uploads/images/d75cc393d06192d8d7a063d5b16acc2b5ed2f676.jpg)图18-JavaScript代码段，显示了和图17中相同的shellcode解密循环

在完成了这个循环后，主要的shellcode主体就解密出来，可以分析了。

在解析了kernel32的基址后，shellcode会解析导出地址表来寻找需要的函数（通过哈希识别）。然后，shellcode会使用LoadLibrary API加载winhttp.dll，并解析这些导出从而找到需要的函数：

| 模块 | 函数(根据哈希导入) |
| :-- | :-- |
| kernel32.dll | CreateThread, WaitForSingleObject, LoadLibraryA, VirtualAlloc, CreateProcessInternalW, GetTempPathW, GetTempFileNameW, WriteFile, CreateFile, CloseHandle |
| winhttp.dll | WinHttpOpen, WinHttpConnect, WinHttpOpenRequest, WinHttpSendRequest, WinHttpReceiveResponse, WinHttpQueryDataAvailable, WinHttpReadData, WinHttpCrackUrl, WinHttpQueryHeaders, WinHttpGetIeProxyConfigForCurrentUser, WinHttpGetProxyForUrl, winHttpSetOption, WinHttpCloseHandle |

如果漏洞利用成功，shellcode就会下载有效载荷并使用前面提到的秘钥解密。Angler会根据不同的利用路径（Internet Explorer, Flash, Silverlight – 至少已知有两个秘钥 ）使用不同的秘钥。

在解析了有效载荷后，shellcode会检查标头来识别有效载荷更像是shellcode（一开始会启动一个什么都不做的NOP函数）还是Windows程序（一开始会识别文本字符串“MZ”）：

![p19](http://drops.javaweb.org/uploads/images/4cace89162fefee71c44d2b049309ffa7cd26ea0.jpg)图19-shellcode中检查有效载荷类型的逻辑

如果解密后的有效载荷是一个程序，那么会保存并运行这个程序。如果这是一个第二阶段的shellcode，那么在这个shellcode（32位和64位都有）的主体中就会内嵌有最后的可执行有效载荷。

当第二阶段的shellcode运行时，有效载荷会被直接插入到漏洞应用进程的内存中，而不会首先写入到磁盘上。这种“[不写文件”的特性](http://malware.dontneedcoffee.com/2014/08/angler-ek-now-capable-of-fileless.html)是Angler最近新增的。这种机制是为了让用户感染Bedep木马家族，这种木马能允许攻击者额外下载其他的木马。

0x04 网络情况
=====

在这一部分，我们会把重点从内容分析转向Angler的网络活动。

**4.1 最新注册**

在大多数web攻击中，Angler使用了最新注册的域名。

在drive-by下载攻击中，我们经常会看到一些注册地址会在短时间内解析到相同的IP。

有时候，Angler也会使用[免费的动态DNS服务](https://nakedsecurity.sophos.com/2012/11/23/hacked-go-daddy-ransomware/)，这是EK广泛使用的一些技术。

**4.2 域名阴影(Domain shadowing)**

Angler也大量使用了[黑来的域名来添加dns记录](https://nakedsecurity.sophos.com/2012/11/23/hacked-go-daddy-ransomware/)，攻击者在好几年的时间中都在使用这种技术，最近这种技术似乎再度流行开来。这些犯罪分子更新了一些合法域名的DNS记录，添加了多个指向恶意EK的子域名-这种技术叫做[域名阴影](http://blogs.cisco.com/security/talos/angler-domain-shadowing)。

图20中就是一个例子，这些活动发生在2015年5月。在这些攻击活动中，使用了更高级的技术，其中DNS记录更新成了通配符项目（`*.foo.example.com`,`*.bar.example.com`）。在这次攻击中，DNS记录更新成了：

*   `*.bro.directorsinstitute.net`
*   `*.fer.directorsinstitute.org`

这样可以允许攻击者把阴影域名解析到恶意IP上，在截稿时，这个IP来自俄罗斯的一台计算机。你可以看到，在Angler重定向中使用的三级域名似乎是一个6字符的字符串，可能是为了允许攻击者跟踪和确定流量的来源（可能是为了统计和付款）。

![p20](http://drops.javaweb.org/uploads/images/9490ad74e9ce5ab6cc7a5a6c03af0de62057018d.jpg)图20-Angler（2015年5月）使用的域名阴影（2级子域名）

在其他的一些例子中，我们发现Angler利用了1级DNS攻击：

![p21](http://drops.javaweb.org/uploads/images/75d711e0bae3b9e58d78029319f0d41296d351b6.jpg)图21-Angler（2015年5月）使用的域名阴影（1级子域名）

在其他一些情况下，子域名中使用的字符串似乎与阴影域名相关。这就说明有人的参与，而不是由程序随机生成的字符串。

域名阴影需要犯罪分子能够修改合法的DNS记录，很可能是通过窃取到的凭证做到的。网站所有者并不都理解DNS记录的关键性，所以很多供应商/注册商也不会关心DNS配置。

大多数网站所有者很少需要更新DNS记录，所以任何更新都需要更多的保护手段，而不仅仅是依靠用户凭证。下面是一些改进建议：

*   实现双重认证
*   在更改了DNS后，发送邮件通知

通过与[Nominet](http://www.nominet.org.uk/)研究人员的合作，我们得以更进一步调查Angler使用的域名阴影活动。我们想要借此确定DNS记录被攻击的时间。我们集中调查了一些DNS查询活动，这些DNS查询活动来自一些与遭到攻击的用户账户相关联的域名，攻击者主要是利用了2级域名阴影来发动了攻击（图20）。

图22中是2015年2月26日的DNS查询活动，包括有3个域名的数据（都是来自同一个遭入侵的账户）。绿圈表示的是总体查询量；紫色方块突出的是超过平均水平的查询量；白圈表示的是快速增加的查询量，当时，攻击者开始在Angler的流量中使用这些域名。

![p22](http://drops.javaweb.org/uploads/images/488458a5f7f5e5c8ad9f83487597fe690d22b148.jpg)图22-3个域名在2015年2月16日的DNS查询时间

通过这些数据可以看出这次攻击有两个阶段。首先，攻击者在早上10点，通过少量的DNS查询测试了DNS记录。这些查询测试的是1级子域名(foo.domain.co.uk)，并且所有的查询都是同一个来源。

一旦攻击者满意域名的解析情况，我们发现大量的恶意Angler流量中使用了2级子域名。在图22中，你可以看到攻击者在围绕使用不同的域名，每个域名只会活动很短的时间。这也符合我们的遥测数据检测，我们也在每个主机名上发现了大量类似的活动。

通过这些信息，我们能更好的了解这些基础设施，以及对用户流量的控制管理。

**4.3 URL 结构**

在Drive-by攻击中的多个组件都使用了一个URL结构，利用这个URL结构可以很好地识别用户流量中的恶意活动。在以前，就有EK在不同的组件中使用了可预见的URL结构，致使安全厂商可以更容易的检测和拦截恶意内容。在某些来自Nuclear和Blackhole EK的样本中包括：

*   时间戳作为文件名
*   文件中使用的哈希和文件夹名称
*   URL的查询部分中使用了有特点的文本

相似的缺陷曾还出现在了早期的Angler中，但是Angler自此以后就在不停地进化，逐步移除了这些弱点，包括在组件使用的容易识别的URL。

0x05 有效载荷
=====

最后这一部分，我们概括了通过Angler安装的木马。为了调查这些木马，我们分析了在2015年4月使用的有效载荷。

你可以看到，所有在这段时间中收集的有效载荷都是通过IE（59%）或Flash（41%）漏洞投放的：

![p23](http://drops.javaweb.org/uploads/images/84d4f6b56cfc37db124f6e6abdc529894da8ce30.jpg)图23-在2015年4月投放Angler有效载荷时利用的漏洞

这一点也符合我们最近找到的Angler登录页数据，我们还没有发现Angler利用了Silverlight或Java漏洞。但是，根据受害者的计算机配置比Angler自己更能说明结果，因为Angler不会漏洞攻击没有安装的组件。

这些drive-by下载活动安装了下列的木马加载：

![p24](http://drops.javaweb.org/uploads/images/2a84e1d94f5c4e447baafb55f20ea9c587663ec9.jpg)图24-Angler在2015年4月安装的木马家族

很明显，这里有一个勒索软件- 有asterisks标签的都是勒索软件家族，占了木马攻击活动的50%以上。最常用的勒索软件是[Teslacrypt](https://nakedsecurity.sophos.com/2015/03/16/teslacrypt-ransomware-attacks-gamers-all-your-files-are-belong-to-us/)。

0x06 总结
=====

在本文中，我们自上而下的分析了Angler EK，强调了Angler如何通过感染web页面来获取更多的流量。

在提供保护时，理解这种行为是非常关键的。Angler会尝试绕过各个 层级上的检测。为了绕过名誉过滤机制，Angler会快速地交换主机名和IP，并使用域名阴影来伪装成合法域名。为了绕过内容检测，Angler会根据受害者来动态地生成组件，并使用各种编码和加密技术。最后，Angler使用了混淆和反沙盒技术来增加样本的收集和分析难度。

如上所述，Angler在近几个月中已经超过了其对手。可能的原因有很多：比如Angler通过感染网页能获取更多的流量，漏洞利用的成功率很高；在犯罪社区中的宣传；更具吸引力的价格-换句话说，从使用Angler按照“安装收费”的回报率更高。

有一件事很明显，对于当今任何浏览网页的用户，Angler都会造成严重的影响。

0x07 致谢
=====

我们感谢安全社区中对追踪EK活动做出的贡献各名用户。但是，我们要致敬[@kafeine](https://twitter.com/kafeine)和[@EKwatcher](https://twitter.com/EKwatcher)对筹备这篇文章所做出的努力。

另外还要感谢SophosLabs的Richard Cohen 和 Andrew O’Donnell对shellcode组件部分做出的贡献。

感谢Nominet 的Ben Taylor, Sion Lloyd 和 Roy Arends 提供的DNS域名阴影技术知识。
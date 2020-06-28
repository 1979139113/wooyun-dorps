# Camera 360应用隐私数据泄露的分析

From:https://www.fireeye.com/blog/threat-research/2015/08/another_popular_andr.html

0x00 前言
-------

* * *

很多流行的Android应用都泄露了隐私数据。我们发现另一款流行的Google Play应用，“Camera 360 Ultimate”,不仅对用户的照片进行了优化，但也不经意间泄露了隐私数据，可以让恶意的用户不经过认证就接触到用户Camera 360的云账户和照片。

在这个发现之前，FireEye的研究者发现了在Camera 360应用和其他应用中大量使用的[SSL协议漏洞](https://www.fireeye.com/blog/threat-research/2014/08/ssl-vulnerabilities-who-listens-when-android-applications-talk.html)。这些漏洞被通过使用中间人攻击的方法利用，并且对用户的隐私构成了严重的威胁。

Android应用开发者应该采取更对的安全措施来为他们的用户提供更安全的手机使用体验。

0x01 概要和介绍
----------

* * *

Camera 360是一个流行的照片拍摄和编辑应用。它在世界范围内有数百万的用户。这个应用会为照片的存储提供免费的云服务。为了使用这些云特色，用户会创建一个可以通过www.cloud.camera360.com访问的云账户。

![](http://drops.javaweb.org/uploads/images/6293c665b85038b445412b8f155789a041594aa1.jpg)

云访问是通过用户名和密码来进行限制的。但是当应用访问晕的时候，它就会通过未加密的形式泄露隐私数据，比如在Android系统日志（logcat）和网络通信过程中。可以读取logcat或者捕获网络通信的应用就可以偷取的这个数据。在你的WiFi网络中的恶意用户也可以通过WiFi嗅探来偷取这个数据。

![](http://drops.javaweb.org/uploads/images/b378816c3c9befdf474309d2db00a9069f226d05.jpg)

泄露的数据可以被用来下载所有用户的照片，除了在那些用户隐私相册中的照片。隐私相册通常都会使用一个额外的密码来保护重要的图像数据。这个应用不会对这些隐私图像进行操作，并且所有从设备上上传的图像都是默认为非隐私的。

0x02 技术细节
---------

* * *

我们分析了Camera 360的最新的版本（6.2）以及先前的版本（6.1.2，6.1.1和6.1），然后在所有的这些版本中都发现了数据泄露。

泄露的数据可以被用来通过如下的步骤来对用户的照片进行未授权访问：

*   通过使用泄露的证书来创建新的登录会话。然后，从服务器获取到图像的密钥并且使用它们来下载图像。
*   劫持登录会话，使用泄露的token来下载图像
*   使用泄露的图像密钥来下载图像而不需要认证

并且，捕获的网络通信内的图像可以被很贱的提取和查看。

以下是所有的细节。

0x03 创建一个登录会话
-------------

* * *

Camera 360应用使用HTTPS登录到服务器，这也就意味着敏感的登录数据不能被轻易的通过网络通信来获取到。在登录的过程中，应用会把隐私数据记录到logcat上去，而这些数据可以被在同一时间运行在这个设备上的其他应用读取到。

Camera 360记录了用户的Email地址，password hash值和其他相关的一些数据。当这些数据泄露的时候，它们就可以被用来创建一个单独的登录会话。作为对登录请求的回应，服务器会返回一个token、用户ID和其他的账户信息。这个token和用户ID可以被用来获取服务器上所有非加密的图像的密钥。利用这些密钥，所有相关的图像都可以被下载下来。

下图展示了我们测试过程中生成的日志信息：

![](http://drops.javaweb.org/uploads/images/caff72c4bb28c56640c4c03fce36975a75abeb35.jpg)

通过对这个应用进行逆向分析，我们发现了他的HTTPS登录URL。在上面提到的日志信息中的数据可以被用来在这个HTTPS请求中创建一个登录会话。这个不带参数的URL如下图所示：

![](http://drops.javaweb.org/uploads/images/16a0d5b6da5d2e6a8e6f5860821076ee84770a3e.jpg)

任何可以读取logcat的应用都可以获得这些登录的数据并且创建它们自己的登录会话。Logcat可以用READ_LOGS权限来读，而这个权限对所有运行在Android4.0和以下的版本上的应用都是可以得到的。但是自从Android4.1（jelly bean）以后，这个权限不再会被授权给第三方应用。

通过逆向这个应用，我们也可以发现**密码的hash值是原始密码的双重MD5并且是unsalted的**。攻击者可以通过使用字典攻击来获取原始的密码，使用彩虹表或者暴力破解来生成一个匹配hash值的字串。破解密码并不是必须的，只要这个hash值可以被直接用来创建登录会话。密码的hash值和窃取到的Email地址可以用来登录camera 360以及云（管理系统）。

0x04 使用泄露的tokens劫持会话
--------------------

* * *

作为对应用登录请求的回应，服务器返回一个token、用户ID和其他账户信息。camera 360会在下一个验证自身的请求中使用这个token和用户ID。

对我们测试账户的服务器响应如下所示：

![](http://drops.javaweb.org/uploads/images/5e783c81487cdabcf01c43c0c40d69f659979ae4.jpg)

这个token是不会过期并且是固定的。它会保持有效即使**用户已经登出**，因为会话变量只是被在客户端删除了而不是服务器端。因此，成功的请求可以在任何时候通过使用这个token来发送。

Camera 360把这些token，以及用户ID、其他应用和设备相关的数据泄漏到了logcat和网络通信上。任何可以读取logcat的Android应用、任何运行在设备上或者在设备的WiFi网络中的网络嗅探器都可以偷取到这些数据。这些泄露的数据可以被用来发送未认证的请求给服务器，也可以笑在云端的所有非隐私图像。

0x05 泄露到logcat的数据
-----------------

* * *

Camera 360会在登录的过程中和用户打开云端账户相关的活动时泄露数据到logcat。

以下是日志信息的两个例子：

![](http://drops.javaweb.org/uploads/images/a88aeb10591284b0c3d002c8ae30f91a4a32acbc.jpg)

在上面的信息中，uid和user Id被设置成了相同的用户ID。token，user token和localkey被设置成了相同的token值。

0x06 泄露到网络通信中的数据
----------------

* * *

这个应用使用HTTPS发送登录请求，但是下一次的请求是通过HTTP发送的，一同发送的还有一个未加密的认证token和user ID。这些未加密的数据可以轻易的从网络通信中读到。

一个这种HTTP请求如下所示：

![](http://drops.javaweb.org/uploads/images/7a46a091082e4eb606bd9340afc533aba908ddbe.jpg)

0x07 使用token和user ID来下载照片
-------------------------

* * *

泄露的token，user ID和其他应用相关的数据可以通过利用以下任何一种请求来获得用户的照片：

![](http://drops.javaweb.org/uploads/images/2e31c5c7772fd3cc6d312417d1b248d8a755a79a.jpg)

这些HTTP请求可以通过两种方法被用来下载照片，如下所示：

**FETCHING IMAGE KEYS**

以上提到的任何HTTP请求都可以被用来从服务器上获得照片的密钥。服务器对我们测试请求的应答如下：

Response for "http://cloud.camera360.com/v2/page/timeline?...."

![](http://drops.javaweb.org/uploads/images/8fb636da3784b5a66f4b446b234b932ebbcac978.jpg)

Response for "http://cloud.camera360.com/v2/page/getNew?..."

![](http://drops.javaweb.org/uploads/images/65c59b6a1cf93101dde64f7f99d9e2f3c022ffec.jpg)

密钥可以从服务器应答中提取，然后使用以下的HTTP请求来下载相关的图像：

![](http://drops.javaweb.org/uploads/images/92ed5682589483774b6b881b09f37feb2803ac33.jpg)

**Bypassing login page of web cloud**

被用来获取图像密钥的HTTP请求同样可以被用来绕过camera 360 云网站的登录（https://cloud.camera360.com/login）。执行任何的这些请求都会让用户登录到web服务，因为这些请求包含了认证token。用户被提示去在一个浏览器标签页中输入这些URLS，然后就直接登录到了云网站的主页。

0x08 使用泄露的照片密钥下载照片
------------------

* * *

camera 360的云相册进程会从服务器上获取最近的照片（非隐私的照片）来向用户展示存储的云照片。它会把接收到的服务器应答记录到logcat上。一个这样的信息如下所示：

![](http://drops.javaweb.org/uploads/images/07d04c973789612e21409af8fd10693aecf20e8b.jpg)

这些记录的密钥可以被那些能够读取logcat的应用窃取到。所有的密钥都由user ID，和一个唯一的照片ID组成。正如上面所提到的，这些密钥可以被用在以下的HTTP请求中来下载图像：

![](http://drops.javaweb.org/uploads/images/17d230cfd4b706de7ad0c4f1cc9100a316b56aac.jpg)

这是一个指向照片的固定链接，并不会失效。这个链接可以被用来在不提供证书或者是认证token的情况下下载照片。

0x09 从捕获的通信中提取照片
----------------

* * *

从网络通信中收集到的这些图像都是未加密并且可以被轻易就看到的。

![](http://drops.javaweb.org/uploads/images/8e24de2ced01a88d8e9cc02d80f31a55241a9703.jpg)

0x0a 预防
-------

* * *

云应用和Android应用安全需要提高，来预防那些更多的数据泄露和未授权数据访问。以下是一些思路：

*   不要在任何的产品中把一些隐私数据记录到Android system log（logcat）上去
*   通过使用以下的几种方法来阻止会话被劫持：
*   不仅仅把登录进程加密，还要把涉及到隐私数据的比如token，userID，照片密钥和照片文件等进行加密
*   对token设置到期时间戳
*   当发送一个登出请求时，最好从服务器端删除掉所有的session variables。不要再次接受以前发布过的token。
*   服务器可以在每个请求中都改变token的值。这样就可以限制攻击者的攻击。
*   token可以和IP地址绑定，但是对那些使用动态IP地址的用户可能不是很方便。
*   指向照片的固定连接应该需要认证，或者让这个链接有时效性。

0x0b 结论
-------

* * *

camera 360在网络通信和Android系统日志中泄露了很多未加密的隐私数据，这会让用户的隐私受到威胁。

FireEye Mobile Threat Prevention Platform可以检测数据泄露和Android应用中已发现的漏洞，并且帮助用户在和应用分享隐私数据方面做出更好的选择。

0x0c Reference
--------------

* * *

http://en.wikipedia.org/wiki/Session_hijacking

http://resources.infosecinstitute.com/session-hijacking-cheat-sheet/
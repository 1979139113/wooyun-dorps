# Android应用安全开发之防范无意识的数据泄露

0x00 简介
=====

OWASP移动安全漏洞Top 10中第4个就是无意识的数据泄漏。当应用程序存储数据的位置本身是脆弱的时，就会造成无意识的数据泄漏。这些位置可能包括剪贴板，URL缓存，浏览器的Cookies，HTML5数据存储，分析数据等等。例如，一个用户在登录银行应用的时候已经把密码复制到了剪贴板，恶意应用程序通过访问用户剪贴板数据就可以获取密码了。

0x01 避免缓存网络数据
=====

数据可以在用户无意识的情况下被各种工具捕获。开发人员经常忽视包括log/debug输出信息，Cookies，Web历史记录，Web缓存等的一些数据存储方式存在的安全隐患。例如，通常浏览器访问页面时，会在临时文件夹下保存页面的html，js，图片等等。当页面上包含敏感信息时，这些信息也会存储在临时文件中。这就造成了安全隐患。在移动设备上尽可能不要存储/缓存敏感数据。这是避免设备上缓存的数据泄漏的最好的方式。

**开发建议**

为了防止HTTP缓存，特别是HTTPS传输数据的缓存，开发人员应该配置Android不缓存网络数据。

为了避免为任何Web过程（如注册）缓存URL历史记录和页面数据，我们应该在Web服务器上配置HTTP缓存头。HTTP协议1.1版中，规定了缓存的使用。其中，Cache-Control: no-store这个应答头可以满足我们的需要。Cache-Control:no-store要求浏览器必须不存储响应或者引起响应的请求的任何内容。对于Web应用程序，HTML表单输入可以通过设置autocomplete=off让浏览器不缓存值。避免缓存应该在应用程序使用后通过对设备数据的取证进行验证。

如果你的应用程序通过WebView访问敏感数据，你可以使用 clearCache()方法来删除任何存储在本地的文件。

0x02 Android:避免GUI对象缓存
=====

由于多任务处理的原因，整个应用程序都可以驻留在内存中，所以Android应用程序界面也会驻留在内存中。发现或者盗取了设备的攻击者可以直接查看到仍然驻留在内存中的用户之前查看过的界面，并看到仍显示在GUI上的所以数据。银行应用程序就是一个例子，一个用户查看了交易记录，然后“退出”应用程序。攻击者通过直接启动交易视图activity可以看到以前的交易被显示出来。

**开发建议**

1.  当用户注销登录的时候退出整个app。这虽然是违反android设计原则的，但是却更加安全，因为GUI界面被销毁、回收了。
2.  在每一个activity（界面）启动的时候检测用户是否处于登录状态，如果没有则跳转到登录界面。
3.  在用户离开（切换）应用界面或者注销登录时清除gui界面的数据

0x03 限制用户名缓存
=====

如果缓存了用户名，在运行时，用户名会在任何类型的身份验证之前加载进内存，从而允许潜在的恶意进程截获用户名。

**开发建议**

很难做到既便利地为用户存储用户名，同时又能避免不安全的存储或潜在的运行时拦截造成的信息泄漏。尽管用户名不像密码那样敏感，但它属于隐私数据应该得到保护。一个安全性较高的缓存用户名的可行的方法就是存储掩蔽的用户名，而不是真实的用户名，如在身份认证的时候用hash值代替用户名。这个hash值可以包含一个唯一的设备token，这个设备token是在用户注册时获取的。使用hash和设备token的好处就是真实的用户名并没有存储在本地，也不会在加载进内存后得不到保护，将这个值复制到其它设备或者在web上使用都会因获取到的设备token值不同而不能使用。攻击者必须挖掘更多的信息（明文帐号、设备特征码、密码）才能成功的窃取用户凭证。

0x04 留意键盘缓存
=====

键盘缓存是意外的数据泄漏问题之一。安卓键盘包含一个用户字典，如果一个用户在文本框输入一些文本，输入法就可能通过用户字典缓存一些由用户输入的数据，用于以后对用户的输入进行自动纠错。而此用户字典不需要什么特殊权限就在任何应用中使用。恶意软件可以通过获取键盘缓存提取这些数据。缓存的内容超出了应用程序的管理权限,所以应用程序不能从缓存中删除数据。

攻击示例：[https://www.youtube.com/watch?v=o6SlUy5mmBQ](https://www.youtube.com/watch?v=o6SlUy5mmBQ)

**开发建议**

对于任何敏感信息（不仅对密码字段）禁用自动纠错的功能。因为键盘缓存的敏感信息可能是可恢复的。

为了提高安全性，可以考虑实现自绘键盘，它可以禁用缓存，并提供其它的保护功能，如键盘监听保护。

0x05 复制和粘贴
=====

无论数据源是否加密，存在于剪贴板中的敏感数据都是可以被任意修改的。如果用户复制的是明文敏感数据，那么其它应用程序通过访问剪贴板就可以获取到该明文敏感数据了。

**修复：**

在适当的情况下，禁用复制/粘贴处理敏感数据。消除复制选项可以减少数据暴露的风险。在安卓系统上，可以通过任何应用程序访问剪贴板，因此，如果需要共享敏感数据，建议使用content provider。

0x06 敏感文件删除
=====

Android通过调用`file.delete()`是不能安全地把文件抹去。只要文件不被覆盖就可以被进行恢复。[Android Data Recovery](http://www.android-recovery.net/android-data-recovery.html)就具备这个功能。

**开发建议**

开发者应该假定写入设备的任何数据都可以被恢复。因此，在某些情况下，加密可以提供额外的一层保护。

另外一种可能方法是删除一个文件，然后创建一个大文件覆盖所有的可用空间，迫使NAND闪存擦除所有未分配空间也是可能的。这种技术的缺点是损耗NAND闪存，导致应用和整个设备的响应速度变慢，显著增加功耗。对于大多数应用不建议使用此方法。理想的解决办法是尽可能不要在设备上存储敏感信息。

0x07 屏幕截取和录制防范
=====

Android 5.0新增的屏幕录制接口，无需特殊权限，使用如下系统API即可实现屏幕录制功能：

![p1](http://drops.javaweb.org/uploads/images/28ce755a66bd01f768d5340723f8104d77a5f47b.jpg)

发起录制请求后，系统弹出如下提示框请求用户确认：

![p2](http://drops.javaweb.org/uploads/images/5f067d9b0b0278a0323939f4b61f8d27c02d1cdc.jpg)

在上图中，“AZ Screen Recorder”为需要录制屏幕的软件名称，“将开始截取您的屏幕上显示的所有内容”是系统自带的提示信息，不可更改或删除。用户点击“立即开始”便开始录制屏幕，录制完成后在指定的目录生成mp4文件。

但其中存在着漏洞，具体参考：[http://www.freebuf.com/vuls/81905.html](http://www.freebuf.com/vuls/81905.html)。攻击者只需要给恶意程序构造一段特殊的，读起来很“合理的”应用程序名，就可以将该提示框变成一个UI陷阱，使其失去原有的“录屏授权”提示功能，并使恶意程序在用户不知情的情况下录制用户手机屏幕。

**开发建议**

在涉及用户隐私的Acitivity中（例如登录，支付等其他输入敏感信息的界面中）增加`WindowManager.LayoutParams.FLAG_SECURE`属性，该属性能防止屏幕被截图和录制。

0x08 参考资料
=====

*   [http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html)
*   [http://resources.infosecinstitute.com/ios-application-security-part-20-local-data-storage-nsuserdefaults-coredata-sqlite-plist-files/](http://resources.infosecinstitute.com/ios-application-security-part-20-local-data-storage-nsuserdefaults-coredata-sqlite-plist-files/)
*   [https://www.nowsecure.com/resources/secure-mobile-development/caching-and-logging/limit-caching-of-username/](https://www.nowsecure.com/resources/secure-mobile-development/caching-and-logging/limit-caching-of-username/)
*   [https://www.nowsecure.com/resources/secure-mobile-development/caching-and-logging/be-aware-of-the-keyboard-cache/](https://www.nowsecure.com/resources/secure-mobile-development/caching-and-logging/be-aware-of-the-keyboard-cache/)
*   [https://www.nowsecure.com/resources/secure-mobile-development/caching-and-logging/be-aware-of-copy-paste/](https://www.nowsecure.com/resources/secure-mobile-development/caching-and-logging/be-aware-of-copy-paste/)
*   [http://www.freebuf.com/vuls/81905.html](http://www.freebuf.com/vuls/81905.html)
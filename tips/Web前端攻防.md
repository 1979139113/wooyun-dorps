# Web前端攻防

0x00 禁止一切外链资源
-------------

* * *

外链会产生站外请求，因此可以被利用实施 CSRF 攻击。

目前国内有大量路由器存在 CSRF 漏洞，其中相当部分用户使用默认的管理账号。通过外链图片，即可发起对路由器 DNS 配置的修改，这将成为国内互联网最大的安全隐患。

**案例演示**

百度旅游在富文本过滤时，未考虑标签的 style 属性，导致允许用户自定义的 CSS。因此可以插入站外资源：

![enter image description here](http://drops.javaweb.org/uploads/images/ec46b0fab2f5fce3ea52e9138d53fa0a4c9b8329.jpg)

所有浏览该页面的用户，都能发起任意 URL 的请求：

![enter image description here](http://drops.javaweb.org/uploads/images/c1f0ea7a3239bc30e1233ff60f7d262281532ea6.jpg)

由于站外服务器完全不受控制，攻击者可以控制返回内容： 如果检测到是管理员，或者外链检查服务器，可以返回正常图片； 如果是普通用户，可以返回 302 重定向到其他 URL，发起 CSRF 攻击。例如修改路由器 DNS：

```
http://admin:admin@192.168.1.1/userRpm/PPPoECfgAdvRpm.htm?wan=0&lcpMru=1480&ServiceName=&AcName=&EchoReq=0&manual=2&dnsserver=黑客服务器&dnsserver2=4.4.4.4&downBandwidth=0&upBandwidth=0&Save=%B1%A3+%B4%E6&Advanced=Advanced

```

![enter image description here](http://drops.javaweb.org/uploads/images/584f30428af5fa253867d793622bffcf3cb49e16.jpg)

演示中，随机测试了几个帖子，在两天时间里收到图片请求 500 多次，已有近 10 个不同的 IP 开始向我们发起 DNS 查询。

![enter image description here](http://drops.javaweb.org/uploads/images/097461298c0635a61e0f2ac5b6aa63cfb90d974e.jpg)

通过中间人代理，用户的所有隐私都能被捕捉到。还有更严重的后果，查考[流量劫持危害探讨](http://fex.baidu.com/blog/2014/04/traffic-hijack-2/)

要是在热帖里『火前留名』，那么数量远不止这些。

如果使用发帖脚本批量回复，将有数以万计的用户网络被劫持。

**防范措施**

杜绝用户的一切外链资源。需要站外图片，可以抓回后保存在站内服务器里。 对于富文本内容，使用白名单策略，只允许特定的 CSS 属性。 尽可能开启 Content Security Policy 配置，让浏览器底层来实现站外资源的拦截。

0x01 富文本前端扫描
------------

* * *

富文本是 XSS 的重灾区。

富文本的实质是一段 HTML 字符。由于历史原因，HTML 兼容众多不规范的用法，导致过滤起来较复杂。几乎所有使用富文本的产品，都曾出现过 XSS 注入。

**案例演示**

旅游发帖支持富文本，我们继续刚才的演示。

![enter image description here](http://drops.javaweb.org/uploads/images/04f90fa8386fad23921c7a166acd5778d9a72d74.jpg)

由于之前已修复过几次，目前只能注入 embed 标签和 src 属性。 但即使这样，仍然可以嵌入一个框架页面：

![enter image description here](http://drops.javaweb.org/uploads/images/84ced49c36d1d3b8b9551b63b85a9733dafcb13f.jpg)

因为是非同源执行的 XSS，所以无法获取主页面的信息。但是可以修改`top.location`，将页面跳转到第三方站点。

将原页面嵌入到全屏的 iframe 里，伪造出相同的界面。然后通过浮层登录框，进行钓鱼。

![enter image description here](http://drops.javaweb.org/uploads/images/f766e34bdcc2eed10234f61677d0254ec75d0f99.jpg)

总之，富文本中出现可执行的元素，页面安全性就大打折扣了。

**防范措施**

这里不考虑后端的过滤方法，讲解使用前端预防方案： 无论攻击者使用各种取巧的手段，绕过后端过滤，但这些 HTML 字符最终都要在前端构造成元素，并渲染出来。

因此可以在 DOM 构造之后、渲染之前，对离屏的元素进行风险扫描。将可执行的元素（script，iframe，frame，object，embed，applet）从缓存中移除。

或者给存在风险的元素，加上沙箱隔离属性。

例如 iframe 加上 sandbox 属性，即可只显示框架内容而不运行脚本 例如 Flash 加上 allowScriptAccess 及 allowNetworking，也能起到一定的隔离作用。

DOM 仅仅被构造是不会执行的，只有添加到主节点被渲染时才会执行。所以这个过程中间，可以实施安全扫描。

实现细节可以参考：[http://www.etherdream.com/FunnyScript/richtextsaferender.html](http://www.etherdream.com/FunnyScript/richtext%5C_safe%5C_render.html)

如果富文本是直接输入到静态页面里的，可以考虑使用 MutationEvent 进行防御。详细参考：[http://fex.baidu.com/blog/2014/06/xss-frontend-firewall-2/](http://fex.baidu.com/blog/2014/06/xss-frontend-firewall-2/)

但推荐使用动态方式进行渲染，可扩展性更强，并且性能消耗最小。

0x02 跳转 opener 钓鱼
-----------------

* * *

浏览器提供了一个 opener 属性，供弹出的窗口访问来源页。但该规范设计的并不合理，导致通过超链接打开的页面，也能使用 opener。

因此，用户点了网站里的超链接，导致原页面被打开的第三方页面控制。

虽然两者受到同源策略的限制，第三方无法访问原页面内容，但可以跳转原页面。

由于用户的焦点在新打开的页面上，所以原页面被悄悄跳转，用户难以觉察到。当用户切回原页面时，其实已经在另一个钓鱼网站上了。

**案例演示**

百度贴吧目前使用的超链接，就是在新窗口中弹出，因此同样存在该缺陷。

攻击者发一个吸引用户的帖子。当用户进来时，引诱他们点击超链接。

通常故意放少部分的图片，或者是不会动的动画，先让用户预览一下。要是用户想看完整的，就得点下面的超链接：

![enter image description here](http://drops.javaweb.org/uploads/images/3a80603eb73f6c7a923385701a4a26075bf0d7c7.jpg)

由于扩展名是 gif 等图片格式，大多用户就毫无顾虑的点了。

事实上，真正的类型是由服务器返回的 MIME 决定的。所以这个站外资源完全有可能是一个网页：

![enter image description here](http://drops.javaweb.org/uploads/images/4e052b4c9cfe2bf28d8bd6d7f6da5acff1b19ebd.jpg)

当用户停留在新页面里看动画时，隐匿其中的脚本已悄悄跳转原页面了。

用户切回原页面时，其实已在一个钓鱼网站上：

![enter image description here](http://drops.javaweb.org/uploads/images/8980d6931b2522e2c4b7272471ca2c19d7706aeb.jpg)

在此之上，加些浮层登录框等特效，很有可能钓到用户的一些账号信息。

**防范措施**

该缺陷是因为 opener 这个属性引起的，所以得屏蔽掉新页面的这个属性。

但通过超链接打开的网页，无法被脚本访问到。只有通过 window.open 弹出的窗口，才能获得其对象。

所以，对页面中的用户发布的超链接，监听其点击事件，阻止默认的弹窗行为，而是用 window.open 代替，并将返回窗体的 opener 设置为 null，即可避免第三方页面篡改了。

详细实现参考：[http://www.etherdream.com/FunnyScript/opener_protect.html](http://www.etherdream.com/FunnyScript/opener_protect.html)

当然，实现中无需上述 Demo 那样复杂。根据实际产品线，只要监听用户区域的超链接就可以。

0x03 用户内容权限
-----------

* * *

支持自定义装扮的场合，往往是钓鱼的高发区。

一些别有用心者，利用装扮来模仿系统界面，引诱用户上钩。

**案例演示 - 空间越界**

百度空间允许用户撰写自定格式的内容。

其本质是一个富文本编辑器，这里不演示 XSS 漏洞，而是利用样式装扮，伪装一个钓鱼界面。

百度空间富文本过滤元素、部分属性及 CSS 样式，但未对 class 属性启用白名单，因此可以将页面上所有的 CSS 类样式，应用到自己的内容上来。

![enter image description here](http://drops.javaweb.org/uploads/images/d9139bac1fa643bd30601e3e5bbde3bf94362f88.jpg)

**防范措施**

规定用户内容尺寸限制，可以在提交时由用户自己确定。

不应该为用户的内容分配无限的尺寸空间，以免恶意用户设置超大字体，破坏整个页面的浏览。

最好将用户自定义的内容嵌套在 iframe 里，以免影响到页面其他部位。

如果必须在同页面，应将用户内容所在的容器，设置超过部分不可见。以免因不可预测的 BUG，导致用户能将内容越界到产品界面上。

**案例演示 - 功能越界**

自定义装扮通常支持站外超链接。

相比贴吧这类简单纯文字，富文本可以将超链接设置在其他元素上，例如图片。

因此这类链接非常具有迷惑性，用户不经意间就点击到。很容易触发之前提到的修改 opener 钓鱼。

![enter image description here](http://drops.javaweb.org/uploads/images/59bed7282e337bf6010742fa10e188b2b6f286a3.jpg)

如果在图片内容上进行伪装，更容易让用户触发一些危险操作。

![enter image description here](http://drops.javaweb.org/uploads/images/c12b3f1813c1ba9c19779e7fab3fd288cca0f17f.jpg)

要是和之前的区域越界配合使用，迷惑性则更强：

![enter image description here](http://drops.javaweb.org/uploads/images/9201826ed73b31f5ea547dc1a6b87ae3eedfad04.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/68d804e29e8f25ec7838cb0e13c93289d11afa63.jpg)

**防范措施**

和之前一样，对于用户提供的超链接，在点击时进行扫描。如果是站外地址，则通过后台跳转进入，以便后端对 URL 进行安全性扫描。

如果服务器检测到是一个恶意网站，或者目标资源是可执行文件，应给予用户强烈的警告，告知其风险。

0x04 点击劫持检测
-----------

* * *

点击劫持算是比较老的攻击方式了，基本原理大家也都听说过。就是在用户不知情的前提下，点击隐藏框架页面里的按钮，触发一些重要操作。

但目前在点击劫持上做防御的并不多，包括百度绝大多数产品线目前都未考虑。

**案例演示**

能直接通过点击完成的操作，比较有意义的就是关注某用户。例如百度贴吧加关注的按钮：

![enter image description here](http://drops.javaweb.org/uploads/images/dc7081ab4936e33c5dd6b96659769aa446af4c4f.jpg)

攻击者事先算出目标按钮的尺寸和坐标，将页面嵌套在自己框架里，并设置框架的偏移，最终只显示按钮：

![enter image description here](http://drops.javaweb.org/uploads/images/2bee0cf1308e00d220914e3b0bc3ada3cac385e6.jpg)

接着通过 CSS 样式，将目标按钮放大，占据整个页面空间，并设置全透明。

![enter image description here](http://drops.javaweb.org/uploads/images/43c686c4b6bc19efc22c52908fb7947c0517ea2e.jpg)

这时虽看不到按钮，但点击页面任意位置，都能触发框架页中加关注按钮的点击：

![enter image description here](http://drops.javaweb.org/uploads/images/d5dc695b4a42be92217f695c5e1c65c51aa8f19a.jpg)

**防范措施**

事实上，点击劫持是很好防御的。

因为自身页面被嵌套在第三方页面里，只需判断 self == top 即可获知是否被嵌套。

对一些重要的操作，例如加关注、删帖等，应先验证是否被嵌套。如果处于第三方页面的框架里，应弹出确认框提醒用户。

确认框的坐标位置最好有一定的随机偏移，从而使攻击者构造的点击区域失效。
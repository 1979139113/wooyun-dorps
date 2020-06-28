# Android勒索软件研究报告

**Author：360移动安全团队**

0x00 摘要
=====

*   手机勒索软件是一种通过锁住用户移动设备，使用户无法正常使用设备，并以此胁迫用户支付解锁费用的恶意软件。其表现为手机触摸区域不响应触摸事件，频繁地强制置顶页面无法切换程序和设置手机PIN码。
*   手机勒索软件的危害除了勒索用户钱财，还会破坏用户数据和手机系统。
*   手机勒索软件最早从仿冒杀毒软件发展演变而来，2014年5月Android平台首个真正意义上的勒索样本被发现。

*   截至2016年第一季度，勒索类恶意软件历史累计感染手机接近90万部，从新增感染手机数据看，2015年第三季度新增感染手机接近35万部。
*   截至2016年第一季度，共捕获手机勒索类恶意样本7.6万余个。其中国外增长迅速，在2015年第三季度爆发式增长，季度捕获量接近2.5万个；国内则稳步上升。
*   国外最常伪装成色情视频、Adobe Flash Player和系统软件更新；国内则最常伪装成神器、外挂及各种刷钻、刷赞、刷人气的软件。
*   从手机勒索软件的技术原理看，锁屏主要利用构造特殊的悬浮窗、Activity劫持、屏蔽虚拟按键、设置手机PIN码和修改系统文件。解锁码生成方式主要是通过硬编码、序列号计算、序列号对应。解锁方法除了直接填写解锁码，还能够通过短信、联网和解锁工具远程控制解锁。
*   制作方面，主要使用合法的开发工具AIDE，通过QQ交流群、教学资料、收徒传授的方式进行指导。
*   传播方面，主要通过QQ群、受害者、贴吧、网盘等方式传播。
*   制马人通过解锁费、进群费、收徒费等方式获取非法所得，日收益在100到300元不等，整个产业链收益可达千万元。
*   从制马人人群特点看，年龄分布呈现年轻化，集中在90后和00后。一方面自己制作勒索软件；另一方面又通过收徒的方式像传销一样不断发展下线。
*   从被敲诈者人人群特点看，主要是一些经常光顾贴吧，以及希望得到各种“利器”、“外挂”的游戏QQ群成员。
*   从预防的角度，可以通过软件大小、名称、权限等方式进行甄别，同时需要提高个人安全意识，养成良好的使用手机习惯。
*   在清除方法上，可以通过重启、360手机急救箱、安全模式、ADB命令、刷机等多种方案解决。

0x01 Android平台勒索软件介绍
=====

### 一、勒索软件定义

手机勒索软件是一种通过锁住用户移动设备，使用户无法正常使用设备，并以此胁迫用户支付解锁费用的恶意软件【1】。

### 二、勒索软件表现形式

**1)**主要通过将手机触摸屏部分或虚拟按键的触摸反馈设置为无效，使触摸区域不响应触摸事件。

![p1](http://drops.javaweb.org/uploads/images/c184590636111e0c3351ee001d1eb052a7a8e462.jpg)

![p2](http://drops.javaweb.org/uploads/images/3b3db2cbe10f46e06847eb82bbd7578d2e9617a1.jpg)

![p3](http://drops.javaweb.org/uploads/images/f282946c5348d16322f34d1e706d890df295407f.jpg)

![p4](http://drops.javaweb.org/uploads/images/aadeb8e59e6952652eb768b91cc8797fba1a80df.jpg)

![p5](http://drops.javaweb.org/uploads/images/6f05727b510380c2296d7d92c6bab3e715294a6c.jpg)

![p6](http://drops.javaweb.org/uploads/images/b0e464213f3d4dca963ab7fde6fc8bb7dd40908a.jpg)

**2)**频繁地强制置顶页面，造成手机无法正常切换应用程序。

![p7](http://drops.javaweb.org/uploads/images/78661611d07dfa51c163cb9326ffb511c84de9d2.jpg)

**3)**设置手机锁定PIN码，无法解锁进入手机系统。

![p8](http://drops.javaweb.org/uploads/images/a8b8fade918679e6d52b53fda19f07b212ee12dc.jpg)

![p9](http://drops.javaweb.org/uploads/images/b92a25c8ef002c4171cd6c3e0098bc0659e0e2b8.jpg)

![p10](http://drops.javaweb.org/uploads/images/5ed90871e1d34c8c90456d795075dcf2fe40bd80.jpg)

![p11](http://drops.javaweb.org/uploads/images/d4e8b39405a8862b1017ec8252ca49af49f9d085.jpg)

### 三、勒索软件的危害

**1)**敲诈勒索用户钱财

**2)**加密手机文件，破坏用户数据

![p12](http://drops.javaweb.org/uploads/images/5dad57cdc2dcc9aeb29e50b33ea41d3068522e3f.jpg)

**3)**清除手机应用，破坏手机系统

![p13](http://drops.javaweb.org/uploads/images/238a8accf3c13ced55a12256a9f953e8f144c3e4.jpg)

### 四、勒索软件历史演变

![p14](http://drops.javaweb.org/uploads/images/db7697a2a640e679d72c99e14bdc066e5ee1a97e.jpg)

*   2013年06月Android.Fakedefender【2】，仿冒杀毒软件恐吓用户手机存在恶意软件要求付费并且顶置付费提示框导致无法清除。
*   2014年05月Android.Koler【3】，Android平台首个真正意义上的勒索样本。
*   2014年06月Android.Simplocker【4】，首个文件加密并且利用洋葱网络的勒索样本。
*   2014年06月Android.TkLocker【5】，360发现首个国内恶作剧锁屏样本。
*   2014年07月Android.Cokri【6】，破坏手机通讯录信息及来电功能的勒索样本。
*   2014年09月Android.Elite【7】，清除手机数据并且群发短信的锁屏样本。
*   2015年05月Android.DevLocker【8】，360发现国内出现首个设置PIN码的勒索样本。
*   2015年09月Android.Lockerpin【9】，国外出现首个设置PIN码的勒索样本。

0x02 Android平台勒索软件现状
=====

### 一、勒索类恶意样本感染量

截至2016年第一季度，勒索类恶意软件历史累计感染手机接近90万部，通过对比2015到2016年季度感染变化趋势，可以看出2015年第三季度新增感染手机接近35万部。

![p15](http://drops.javaweb.org/uploads/images/c81e544f386bfbfd3e7b7ca03470a722d01654b8.jpg)

### 二、勒索类恶意样本数量

截至2016年第一季度，共捕获勒索类恶意样本7.6万余个，通过对比2015到2016年季度变化情况，可以看出国外勒索类恶意软件增长迅速，并且在2015年第三季度爆发式增长，季度捕获量接近2.5万个；反观国内勒索类恶意软件增长趋势，虽然没有爆发式增长，但是却呈现出稳步上升的趋势。

![p16](http://drops.javaweb.org/uploads/images/446819fdc1dee1826505ae5831ead67c2e1e1f7e.jpg)

### 三、常见伪装对象

对比国内外勒索类恶意软件最常伪装的软件名称可以看出，国外勒索类恶意软件最常伪装成色情视频、Adobe Flash Player和系统软件更新这类软件。而国内勒索类恶意软件最常伪装成神器、外挂及各种刷钻、刷赞、刷人气的软件，这类软件往往利用了人与人之间互相攀比的虚荣心和侥幸心理。

![p17](http://drops.javaweb.org/uploads/images/af70220bd70d147f710fc835335b79867ca860bd.jpg)

### 四、用户损失估算

2015年全年国内超过11.5万部用户手机被感染，2016年第一季度国内接近3万部用户手机被感染。每个勒索软件的解锁费用通常为20、30、50元不等，按照每位用户向敲诈者支付30元解锁费用计算，2015年国内用户因此遭受的损失达到345万元，2016年第一季度国内用户因此遭受的损失接近90万。

0x03 Android平台勒索软件技术原理
=====

### 一、锁屏技术原理

**1)**利用WindowManager.LayoutParams的flags属性

通过addView方法实现一个悬浮窗，设置WindowManager.LayoutParams的flags属性，例如，“`FLAG_FULLSCREEN`”、“`FLAG_LAYOUT_IN_SCREEN`”配合“`SYSTEM_ALERT_WINDOW`”的权限，使这个悬浮窗全屏置顶且无法清除，造成手机屏幕锁屏无法正常使用。

![p18](http://drops.javaweb.org/uploads/images/4e23ca5db342246803f5239fd3003f3ec20b3fce.jpg)

**2)**利用Activity劫持

通过TimerTask定时监控顶层Activity，如果检测到不是自身程序，便会重新启动并且设置addFlags值为“`FLAG_ACTIVITY_NEW_TASK`”覆盖原来程序的Activity，从而利用Activity劫持手段，达到勒索软件页面置顶的效果，同时结束掉原来后台进程。目前Android5.0以上的版本已经采取了保护机制来阻止这种攻击方式，但是5.0以下的系统仍然占据绝大部分。

![p19](http://drops.javaweb.org/uploads/images/c7643f5e689e8eec748b83985af1716f066b60ac.jpg)

**3)**屏蔽虚拟按键

通过改写onKeyDown方法，屏蔽返回键、音量键、菜单键等虚拟按键，造成不响应按键动作的效果，来达到锁屏的目的。

![p20](http://drops.javaweb.org/uploads/images/0ab7f377e755d9b2de43718d5dbd3ac8bbcd0577.jpg)

**4)**利用设备管理器设置解锁PIN码

通过诱导用户激活设备管理器，勒索软件会在用户未知情的情况下强制给手机设置一个解锁PIN码，导致用户无法解锁手机。

![p21](http://drops.javaweb.org/uploads/images/152727533c3ddfffe9ba67781bc1ba18572f5739.jpg)

**5)**利用Root权限篡改系统文件

如果手机之前设置了解锁PIN码，勒索软件通过诱导用户授予Root权限，篡改/data/system/password.key文件，在用户不知情的情况下设置新的解锁PIN码替换旧的解锁PIN码，达到锁屏目的。

![p22](http://drops.javaweb.org/uploads/images/acf8bcf50ee77db9e87e814d82af5f94cf02d531.jpg)

### 二、解锁码生成方式

**1)**解锁码硬编码在代码里

有些勒索软件将解锁码硬编码在代码里，这类勒索软件解锁码唯一，且没有复杂的加密或计算逻辑，很容易找到解锁码，比较简单。

![p23](http://drops.javaweb.org/uploads/images/0d054e9c39659172a1392e5b0fa55fa8479b71ea.jpg)

**2)**解锁码通过序列号计算

与硬编码的方式相比，大部分勒索软件在页面上都显示了序列号，它是恶意软件作者用来标识被锁住的移动设备编号。一部分勒索软件解锁码是通过序列号计算得出，例如下图中的fkey代表序列号，是一个随机生成的数；key代表解锁码，解锁码是序列号*3-98232计算得出。这仅是一个简单的计算例子，这种方式解锁码随序列号变化而变化，解锁码不唯一并且可以使用复杂的计算逻辑。

![p24](http://drops.javaweb.org/uploads/images/c837510ebf39e3ea52c96a524dcf2dc5e6f41217.jpg)

**3)**解锁码与序列号键值对关系

还有一部分勒索软件，解锁码与序列号都是随机生成，使用键值对的方式保留了序列号和解锁码的对应关系。这种方式序列号与解锁码没有计算关系，解锁码会经过各种加密变换，通过邮件等方式回传解锁码与序列号的对应关系。

![p25](http://drops.javaweb.org/uploads/images/0b335762f633aa42f125efc6d9c0fa54f139f15e.jpg)

### 三、常见解锁方法

**1)**直接输入解锁码解锁

用户通过付给敲诈者钱来换取设备的解锁码。将解锁码直接输入在勒索页面里来解锁屏幕，这是最常见的勒索软件的解锁方式之一。

**2)**利用短信控制解锁

短信控制解锁方式，就是通过接收指定的短信号码或短信内容远程解锁，这种解锁方式会暴露敲诈者使用的手机号码。

![p26](http://drops.javaweb.org/uploads/images/439638cfb773a53683636c3d518c1954f7f9168a.jpg)

**3)**利用联网控制解锁

敲诈者为了隐藏自身信息，会使用如洋葱网络等匿名通信技术远程控制解锁。这种技术最初是为了保护消息发送者和接受者的通信隐私，但是被大量的恶意软件滥用。

![p27](http://drops.javaweb.org/uploads/images/6e6e697431c43737c86766047fba561de468635f.jpg)

**4)**利用解锁工具解锁

敲诈者为了方便进行勒索，甚至制作了勒索软件配套的解锁控制端。

![p28](http://drops.javaweb.org/uploads/images/871ea2d80262b62043dd133391f49a49865ed8bb.jpg)

0x04 Android平台勒索软件黑色产业链
=====

本章主要从Android平台勒索软件的制作、传播、收益角度及制马人和被敲诈的人群特点，重点揭露其在国内的黑色产业链。

### 一、制作

**（一）制作工具**

国内大量的锁屏软件都使用合法的开发工具AIDE，AIDE是Android环境下的开发工具，这种开发工具不需要借助电脑，只需要在手机上操作，便可以完成Android样本的代码编写、编译、打包及签名全套开发流程。制马人使用开发工具AIDE只需要对源码中代表QQ号码的字符串进行修改，便可以制作成一个新的勒索软件。

因为这种工具操作简单方便，开发门槛低，变化速度快，使得其成为制马人开发勒索软件的首选。

![p29](http://drops.javaweb.org/uploads/images/d9cd1f52af97fddde8e66c25ff783a8d74b32716.jpg)

**（二）交流群**

通过我们的调查发现，制马人大多使用QQ群进行沟通交流，在QQ群查找里输入“Android锁机”等关键字后，就能够找到很多相关的交流群。

![p30](http://drops.javaweb.org/uploads/images/02190820d2e7285f067d1a9626aaa9740b41a916.jpg)

图是某群的群主在向群成员炫耀自己手机中保存的锁机源码

![p31](http://drops.javaweb.org/uploads/images/83ea5af173fc0c59bf8ac03d9b8fac29bbfad287.jpg)

**（三）教学资料**

制作时不仅有文字资料可以阅读，同时在某些群里还提供了视频教程，可谓是“图文并茂”

锁机教程在线视频

![p32](http://drops.javaweb.org/uploads/images/e5c58afb61368ee12f321386a727f379fda3018c.jpg)

锁机软件教程

![p33](http://drops.javaweb.org/uploads/images/018598b4d5e50224c2afe21be6565ab1b739fa9e.jpg)

**（四）收徒传授**

在群里，群主还会以“收徒”的方式教授其他人制作勒索软件，在扩大自己影响力的同时，也能够通过这种方式获取利益。

![p34](http://drops.javaweb.org/uploads/images/36a6c3e96682435e39215459a65e3a7c572f5d22.jpg)

### 二、传播

通过我们的调查研究，总结出了国内勒索软件传播示意图

![p35](http://drops.javaweb.org/uploads/images/c162e5c1107b552594f9b24c5811313e15bf9ea8.jpg)

制马人通过QQ群、受害者、贴吧、网盘等方式，来传播勒索软件。

**（一）QQ群**

制马人通过不断加入各种QQ群，在群共享中上传勒索软件，以“外挂”、“破解”、“刷钻”等各种名义诱骗群成员下载，以达到传播的目的。

![p36](http://drops.javaweb.org/uploads/images/fc12a0ee5511e693668cf4de22030128a2eafa56.jpg)

**（二）借助受害者**

当有受害者中招时，制马人会要求受害者将勒索软件传播到更多的QQ群中，以作为换取解锁的条件。

![p37](http://drops.javaweb.org/uploads/images/c5e4d9bb44d7fffa7763921cde96b7e581b3256d.jpg)

**（三）贴吧**

制马人在贴吧中以链接的方式传播。

![p38](http://drops.javaweb.org/uploads/images/8193d5b5b3f9bc9716ccd1490b1ab065fb082e56.jpg)

**（四）网盘**

制马人将勒索软件上传到网盘中，再将网盘链接分享到各处，以达到传播的目的。

![p39](http://drops.javaweb.org/uploads/images/72fc99b4609e42880b521255e12c923bd0011934.jpg)

### 三、收益

通过我们的调查研究，总结出了国内勒索软件产业链的资金流向示意图

![p40](http://drops.javaweb.org/uploads/images/c409d97c26387f06c51a2f8cd6b7b14c4f07598c.jpg)

制马人主要通过解锁费、进群费、收徒费等方式获取非法所得，日收益在100到300元不等。

**（一）收益来源**

解锁费

![p41](http://drops.javaweb.org/uploads/images/2367ba5dd7364151b1582b123fbbc222ba83cbee.jpg)

进群费

![p42](http://drops.javaweb.org/uploads/images/cca6fd5e802130c5314df6b31493fa0c7c6696c1.jpg)

收徒费

![p43](http://drops.javaweb.org/uploads/images/a0cf7a5c690d4167b59b6074884f1650f7e4884d.jpg)

**（二）日均收益**

制马人通过勒索软件的日收益在100到300元不等

![p44](http://drops.javaweb.org/uploads/images/5cf1551236645b125435ac7ba7c5da19ad104c4c.jpg)

![p45](http://drops.javaweb.org/uploads/images/ed32f1beab54363aef6a4946f209e70c1b678ba1.jpg)

**（三）产业链收益**

2015年全年国内超过11.5万部用户手机被感染，2016年第一季度国内接近3万部用户手机被感染。每个勒索软件的解锁费用通常为20、30、50元不等，按照每个勒索软件解锁费用30元计算，2015年国内Android平台勒索类恶意软件产业链年收益达到345万元，2016年第一季度接近90万。国内Android平台勒索类恶意软件历史累计感染手机34万部，整个产业链收益超过了千万元，这其中还不包括进群和收徒费用的收益。

### 四、制马人人群特点

**（一）制马人年龄分布**

从抽取的几个传播群中的人员信息可以看出，制马人的年龄分布呈现年轻化，集中在90后和00后。

![p46](http://drops.javaweb.org/uploads/images/5e1f93ca4bf1986a01ee80da235bb5374cb6d01a.jpg)

**（二）制马人人员架构**

绝大多数制马人既扮演着制作者，又扮演着传播者的角色。他们一方面自己制作勒索软件，再以各种方式进行传播；另一方面又通过收徒的方式像传销一样不断发展下线，使制马人和传播者的人数不断增加，勒索软件的传播范围更广。

![p47](http://drops.javaweb.org/uploads/images/50592bef71962b90e057408fa23a4640205d85fd.jpg)

这群人之所以肆无忌惮制作、传播勒索软件进行勒索敲诈，并且大胆留下自己的QQ、微信以及支付宝账号等个人联系方式，主要是因为他们年龄小，法律意识淡薄，认为涉案金额少，并没有意识到触犯法律。甚至以此作为赚钱手段，并作为向他人进行炫耀的资本。

### 五、被敲诈人人群特点

通过一些被敲诈的用户反馈，国内敲诈勒索软件感染目标人群，主要是针对一些经常光顾贴吧的人，以及希望得到各种“利器”、“外挂”的游戏QQ群成员。这类人绝大多数是90后或00后用户，抱有不花钱使用破解软件或外挂的侥幸心理，或者为了满足互相攀比的虚荣心，容易被一些带有“利器”、“神器”、“刷钻”、“刷赞”、“外挂”等名称的软件吸引，从而中招。

0x05 Android平台勒索软件的预防
=====

### 一、勒索软件识别方法

**1)软件大小**

安装软件时观察软件包的大小，这类勒索软件都不会太大，通常不会超过1M。

**2)软件名称**

多数勒索软件都会伪装成神器、外挂及各种刷钻、刷赞、刷人气的软件。

**3)软件权限**

多数勒索软件会申请“SYSTEM_ALERT_WINDOW”权限或者诱导激活设备管理器，需要在安装和使用时留意。

### 二、提高个人安全意识

**1)可信软件源**

建议用户在选择应用下载途径时，应该尽量选择大型可信站点，如360手机助手、各软件官网等。

**2)安装安全软件**

建议用户手机中安装安全软件，实时监控手机安装的软件，如360手机卫士。

**3)数据备份**

建议用户日常定期备份手机中的重要数据，比如通讯录、照片、视频等，避免手机一旦中招，给用户带来的巨大损失。

**4)拒绝诱惑**

建议用户不要心存侥幸，被那些所谓的能够“外挂”、“刷钻”、“破解”软件诱惑，这类软件绝大部分都是假的，没有任何功能，只是为了吸引用户中招。

**5)正确的解决途径**

一旦用户不幸中招，建议用户不要支付给敲诈者任何费用，避免助涨敲诈者的嚣张气焰。用户可以向专业的安全人员或者厂商寻求解决方法。

0x06 Android平台勒索软件清除方案
=====

### 一、手机重启

手机重启后快速对勒索软件进行卸载删除是一种简单便捷的清除方法，但这种方法取决于手机运行环境和勒索软件的实现方法，仅可以对少部分勒索软件起作用。

### 二、360手机急救箱

360手机急救箱独有三大功能：“安装拦截”、“超强防护”、“摇一摇杀毒”，可以有效的查杀勒索软件。

安装拦截功能，可以让勒索软件无法进入用户手机；

超强防护功能，能够清除勒索软件未经用户允许设置的锁屏PIN码，还能自动卸载木马；

摇一摇杀毒可以在用户中了勒索软件，无法操作手机的情况下，直接杀掉木马，有效保护用户安全。

### 三、安全模式

安全模式也是一种有效的清除方案，不同的机型进入安全模式的方法可能不同，建议用户查找相应机型进入安全模式的具体操作方法。

我们以Nexus 5为例介绍如何进入安全模式清除勒索软件，供用户参考。步骤如下：

*   步骤一：当手机被锁后长按手机电源键强制关机，然后重启手机。
*   步骤二：当出现Google标志时，长按音量“-”键直至进入安全模式。
*   步骤三：进入“设置”页面，找到并点击“应用”。
*   步骤四：找到对应的恶意应用，点击卸载，重启手机，清除成功。

![p48](http://drops.javaweb.org/uploads/images/ba96d063d61c48d5ef9628f0ba5e811b3b6e642c.jpg)

### 四、ADB命令

对有一定技术基础的用户，在手机有Root权限并且已经开启USB调试（设置->开发者选项->USB调试）的情况下，可以将手机连接到电脑上，通过ADB命令清除勒索软件。

针对设置PIN码类型的勒索软件，需要在命令行下执行以下命令：

```
> adb shell
> su
> rm /data/system/password.key

```

针对其他类型的勒索软件，同样需要在命令行下执行rm命令，删除勒索软件安装的路径。

### 五、刷机

如以上方法都无法解决，用户参考手机厂商的刷机指导或者到手机售后服务，在专业指导下进行刷机操作。

0x07 附录：参考资料
=====

*   【1】：[Mobile security](https://en.wikipedia.org/wiki/Mobile_security#Ransomware)
*   【2】：[FakeAV holds Android Phones for Ransom](http://www.symantec.com/connect/blogs/fakeav-holds-android-phones-ransom)
*   【3】：[Police Locker land on Android Devices](http://malware.dontneedcoffee.com/2014/05/police-locker-available-for-your.html)
*   【4】：[ESET Analyzes Simplocker – First Android File-Encrypting, TOR-enabled Ransomware](http://www.welivesecurity.com/2014/06/04/simplocker/)
*   【5】：[TkLocker分析报告](http://blogs.360.cn/360mobile/2014/06/18/analysis_of_tklocker/)
*   【6】：[病毒播报之流氓勒索](http://blog.avlyun.com/2014/07/1295/%E7%97%85%E6%AF%92%E6%92%AD%E6%8A%A5%E4%B9%8B%E6%B5%81%E6%B0%93%E5%8B%92%E7%B4%A2/)
*   【7】：[Vandal Trojan for Android wipes memory cards and blocks communication](http://news.drweb.com/show/?i=5978&c=9&lng=en&p=0)
*   【8】：[手机锁屏勒索国内首现身](http://blogs.360.cn/360mobile/2015/05/19/analysis_of_ransomware/)
*   【9】：[Aggressive Android ransomware spreading in the USA](http://www.welivesecurity.com/2015/09/10/aggressive-android-ransomware-spreading-in-the-usa/)
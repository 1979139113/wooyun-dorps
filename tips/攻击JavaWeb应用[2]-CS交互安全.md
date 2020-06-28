# 攻击JavaWeb应用[2]-CS交互安全

> 注:本节意在让大家了解客户端和服务器端的一个交互的过程,我个人不喜欢xss,对xss知之甚少所以只能简要的讲解下。这一节主要包含HttpServletRequest、HttpServletResponse、session、cookie、HttpOnly和xss,文章是年前几天写的本应该是有续集的但年后就没什么时间去接着续写了。由于工作并非安全行业，所以写的并不算专业希望大家能够理解。后面的章节可能会有Java里的SQL注入、Servlet容器相关、Java的框架问题、eclipse代码审计等。


### 0x00 Request & Response(请求与响应)

* * *

请求和响应在Web开发当中没有语言之分不管是ASP、PHP、ASPX还是JAVAEE也好，Web服务的核心应该是一样的。

在我看来Web开发最为核心也是最为基础的东西就是Request和Response！我们的Web应用最终都是面向用户的，而请求和响应完成了客户端和服务器端的交互。服务器的工作主要是围绕着客户端的请求与响应的。

如下图我们通过Tamper data拦截请求后可以从请求头中清晰的看到发出请求的客户端请求的地址为：localhost。

浏览器为FireFox，操作系统为Win7等信息，这些是客户端的请求行为，也就是Request。 ￼ 

![enter image description here](http://drops.javaweb.org/uploads/images/e39a225152fb4ccc0b89f5022e71c93b9c18efa7.jpg)

当客户端发送一个Http请求到达服务器端之后，服务器端会接受到客户端提交的请求信息(HttpServletRequest)，然后进行处理并返回处理结(HttpServletResopnse)。

下图演示了服务器接收到客户端发送的请求头里面包含的信息： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/0c6cb08ba539a6c23d4b0f1c66023e7fd61ddf7c.jpg)

  页面输出的内容为：

```
 host=localhost
 user-agent=Mozilla/5.0 (Windows NT 6.1; rv:18.0) Gecko/20100101 Firefox/18.0 
accept=text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8 
accept-language=zh-cn,zh;q=0.8,en-us;q=0.5,en;q=0.3 
accept-encoding=gzip, deflate 
connection=keep-alive

```

#### 请求头信息伪造XSS

关于伪造问题我是这样理解的:发送Http请求是客户端的主动行为，服务器端通过ServerSocket监听并按照Http协议去解析客户端的请求行为。

所以请求头当中的信息可能并不一定遵循标准Http协议。

用FireFox的Tamper Data和Moify Headers（FireFox扩展中心搜Headers和Tamper Data都能找到） 插件修改下就实现了，请先安装FireFox和Tamper Data：  ￼    点击Start Tamper 然后请求Web页面，会发现请求已经被Tamper Data拦截下来了。选择Tamper：

![enter image description here](http://drops.javaweb.org/uploads/images/eb02d6840ab41c4001d63111be57c24d54552bee.jpg)

点击Start Tamper 然后请求Web页面，会发现请求已经被Tamper Data拦截下来了。选择Tamper：

![enter image description here](http://drops.javaweb.org/uploads/images/3e8bc69281cddc74d83551a861c2790ac02f1e40.jpg)

修改请求头信息： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/ddcd95f5e4fab62fd456b55c1a9448a595d4329c.jpg)

 Servlet Request接受到的请求：

```
Enumeration e = request.getHeaderNames();
while (e.hasMoreElements()) {
    String name = (String) e.nextElement();//获取key
    String value = request.getHeader(name);//得到对应的值
    out.println(name + "=" + value + "<br>");//输出如cookie=123
}     

```

![enter image description here](http://drops.javaweb.org/uploads/images/3823204f4e94c566f1a20b5c49d9378d09e7eb5b.jpg)

源码下载：[http://pan.baidu.com/share/link?shareid=166499&uk=2332775740](http://pan.baidu.com/share/link?shareid=166499&uk=2332775740)

使用Moify Headers自定义的修改Headers:

![enter image description here](http://drops.javaweb.org/uploads/images/1f5b65ad1268b2cf22bf9a407531b6872851cfbb.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/5a987c00240171d27981f2ac1addd9dd4295047c.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/a5a5ec60486931520e3d6db3580729286d757207.jpg)

修改请求头的作用是在某些业务逻辑下程序猿需要去记录用户的请求头信息到数据库，而通过伪造的请求头一旦到了数据库可能造成xss，或者在未到数据库的时候就造成了SQL注入，因为对于程序员来说，大多数人认为一般从Headers里面取出来的数据是安全可靠的，可以放心的拼SQL(记得好像Discuz有这样一个漏洞)。今年一月份的时候我发现xss.tw也有一个这样的经典案例，Wdot那哥们在记录用户的请求头信息的时候没有去转意特殊的脚本，导致我们通过伪造的请求头直接存储到数据库。

XSS.tw平台由于没有对请求头处理导致可以通过XSS屌丝逆袭高富黑。 刚回来的时候被随风玩爆菊了。通过修改请求头信息为XSS脚本，xss那平台直接接收并信任参数，因为很少有人会蛋疼的去怀疑请求头的信息，所以这里造成了存储型的XSS。只要别人一登录xss就会自动的执行我们的XSS代码了。

Xss.tw由于ID很容易预测，所以很轻易的就能够影响到所有用户：

￼![enter image description here](http://drops.javaweb.org/uploads/images/89aec4334062595b80ef010736f6ba7e8fc7cbc9.jpg)

于是某一天就有了所有的xss.tw用户被随风那2货全部弹了www.gov.cn: ￼

![enter image description here](http://drops.javaweb.org/uploads/images/b4ff764a5394451b29144ac961103e70ea75e57c.jpg)

#### Java里面伪造Http请求头

代码就不贴了，在发送请求的时候设置setRequestProperty 就行了，如：

```
URL realUrl = new URL(url);
URLConnection connection = realUrl.openConnection();
connection.setConnectTimeout(5000);//连接超时
connection.setReadTimeout(5000);// 读取超时
connection.setRequestProperty("accept", "*/*");
connection.setRequestProperty("connection", "Keep-Alive");
(………………………..)

```

![enter image description here](http://drops.javaweb.org/uploads/images/d88ad3ae56f828840d23512160d196695ac55ba1.jpg)

Test Servlet:

![enter image description here](http://drops.javaweb.org/uploads/images/0c97061814a0d211b6b158a4783af5dc229bbd27.jpg)￼

### 0x01 Session

* * *

Session是存储于服务器内存当中的会话，我们知道Http是无状态协议，为了支持客户端与服务器之间的交互，我们就需要通过不同的技术为交互存储状态，而这些不同的技术就是Cookie和Session了。

设置一个session:

```
session.setAttribute("name",name);//从请求中获取用户的name放到session当中
 session.setAttribute("ip",request.getRemoteAddr());//获取用户请求Ip地址
out.println("Session 设置成功.");

```

![enter image description here](http://drops.javaweb.org/uploads/images/cab10452f2575e934b0e91aa8d796b1e57cdbe0a.jpg)

直接获取session如下图可以看到我们用FireFox和Chrome请求同一个URL得到的SessionId并不一样，说明SessionId是唯一的。一旦Session在服务器端设置成功那么我们在此次回话当中就可以一直共享这个SessionId对应的session信息，而session是有有效期的，一般默认在20-30分钟，你会看到xss平台往往有一个功能叫keepSession，每过一段时间就带着sessionId去请求一次，其实就是在保持session的有效不过期。

![enter image description here](http://drops.javaweb.org/uploads/images/d169124f8a84ebba4c4bc97edb3f2db27066cc67.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/9ac3ffcfceec3ac46c4f3dadd3288c8e107cd6e0.jpg)

#### Session 生命周期(从创建到销毁)

  1、session的默认过期时间是30分钟，可修改的最大时间是1440分钟（1440除以60=24小时=1天）。

 2、服务器重启或关闭Session失效。

#####  注：浏览器关闭其实并不会让session失效！因为session是存储在服务器端内存当中的。客户端把浏览器关闭了服务器怎么可能知道？正确的解释或许应该是浏览器关闭后不会去记忆关闭前客户端和服务器端之间的session信息且服务器端没有将sessionId以Cookie的方式写入到客户端缓存当中，重新打开浏览器之后并不会带着关闭之前的sessionId去访问服务器URL，服务器从请求中得不到sessionId自然给人的感觉就是session不存在（自己理解的）。

当我们关闭服务器时Tomcat会在安装目录workCatalinalocalhost项目名目录下建立SESSIONS.ser文件。此文件就是Session在Tomcat停止的时候 持久化到硬盘中的文件. 所有当前访问的用户Session都存储到此文件中. Tomcat启动成功后.SESSIONS.ser  又会反序列化到内存中,所以启动成功后此文件就消失了. 所以正常情况下 从启Tomcat用户是不需要登录的. 注意有个前提，就是存储到Session里面的user对象所对应的User类必须要序列化才可以。（摘自：[http://alone-knight.iteye.com/blog/1611112](http://alone-knight.iteye.com/blog/1611112)）

#### SessionId是神马？有什么用？

  我们不妨来做一个偷取sessionId的实验： 首先访问：http://localhost/Test/SessionTest?action=setSession&name=selina 完成session的创建，如何建立就不解释了如上所述。 

同时开启FireFox和Chrome浏览器设置两个Session：  ￼   

![enter image description here](http://drops.javaweb.org/uploads/images/902635d2d2a2a0ab91e4d32369b32595c36078f0.jpg)

我们来看下当前用户的请求头分别是怎样的：  ￼ 

![enter image description here](http://drops.javaweb.org/uploads/images/c7f82970bb345f5c971c917091ab1e0a790514bb.jpg)

  我们依旧用TamperData来修改请求的Cookie当中的jsessionId，下面是见证奇迹的时刻：  ￼

![enter image description here](http://drops.javaweb.org/uploads/images/1f87485cf8ff768e2aab1cb07b089f961f8d5b49.jpg)

我要说的话都已经在图片当中的文字注释里面了，伟大的Xss黑客们看明白了吗？你盗取的也许是jsessionId(Java里面叫jsessionId)，而不只是cookie。那么假设我们的Session被设置得特别长那么这个SessionId就会长时间的保留，而为Xss攻击提供了得天独厚的条件。而这种Session长期存在会浪费服务器的内存也会导致：SessionFixation攻击！  

#### 如何应对SessionFixation攻击

  1、用户输入正确的凭据，系统验证用户并完成登录，并建立新的会话ID。 

2、Session会话加Ip控制 

3、加强程序员的防范意识：写出明显xss的程序员记过一次，写出隐晦的xss的程序员警告教育一次，连续查出存在3个及其以上xss的程序员理解解除劳动合同(哈哈，开玩笑了)。

### 0x02 Cookie

* * *

Cookie是以文件形式[缓存在客户端]的凭证(精简下为了通俗易懂)，cookie的生命周期主要在于服务器给设置的有效时间。如果不设置过期时间，则表示这个cookie生命周期为浏览器会话期间，只要关闭浏览器窗口，cookie就消失了。 这次我们以IE为例：

![enter image description here](http://drops.javaweb.org/uploads/images/901cc96942e544600bfd05079a9b4ce160200c4f.jpg)

 我们来创建一个Cookie：  

```
if(!"".equals(name)){
            Cookie cookies = new Cookie("name",name);//把用户名放到cookie
            cookies.setMaxAge(60*60*60*12*30) ;//设置cookie的有效期
     //     c1.setDomain(".ahack.net");//设置有效的域     
       response.addCookie(cookies);//把Cookie保存到客户端
            out.println("当前登录:"+name);
}else {
            out.println("用户名不能为空!");     
}

```

有些大牛级别的程序员直接把帐号密码明文存储到客户端的cookie里面去，不得不佩服其功力深厚啊。客户端直接记事本打开就能看到自己的帐号密码了。

![enter image description here](http://drops.javaweb.org/uploads/images/814fed30ac4ca8911f7cb0e8f5da4e0ac0b84202.jpg)

 继续读取Cookie：

![enter image description here](http://drops.javaweb.org/uploads/images/76686133956662de5c3881ff4a0b0c90f9398e8d.jpg)

我想cookie以明文的形式存储在客户端我就不用解释了吧？文件和数据摆在面前！ 盗取cookie的最直接的方式就是xss，利用IE浏览器输出当前站点的cookie：

```
javascript:document.write(document.cookie)          ￼

```

![enter image description here](http://drops.javaweb.org/uploads/images/f235192a1062074b7c28b63a1991e6667cf01225.jpg)

  首先我们用FireFox创建cookie： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/0054cd3363f39f732039284b8140f460e400f4ad.jpg)

然后TamperData修改Cookie：

![enter image description here](http://drops.javaweb.org/uploads/images/5b4289b9fb5c769cc28ce225bfc7ccdc45ef88d2.jpg)

一般来说直接把cookie发送给服务器服务器，程序员过度相信客户端cookie值那么我们就可以在不用知道用户名和密码的情况下登录后台，甚至是cookie注入。jsessionid也会放到cookie里面，所以拿到了cookie对应的也拿到了jsessionid，拿到了jsessionid就拿到了对应的会话当中的所有信息，而如果那个jsessionid恰好是管理员的呢？

### 0x03 HttpOnly

* * *

上面我们用

```
javascript:document.write(document.cookie)

```

通过document对象能够拿到存储于客户端的cookie信息。

HttpOnly设置后再使用document.cookie去取cookie值就不行了。

通过添加HttpOnly以后会在原cookie后多出一个HttpOnly;

普通的cookie设置：

```
Cookie: jsessionid=AS348AF929FK219CKA9FK3B79870H;

```

加上HttpOnly后的Cookie：

```
Cookie: jsessionid=AS348AF929FK219CKA9FK3B79870H; HttpOnly;

```

（参考YearOfSecurityforJava）

在JAVAEE6的API里面已经有了直接设置HttpOnly的方法了： ￼![enter image description here](http://drops.javaweb.org/uploads/images/c13dd77ca487bc6651dce502df5eda7dd008b7a2.jpg)

API的对应说明：

大致的意思是：如果isHttpOnly被设置成true，那么cookie会被标识成HttpOnly.能够在一定程度上解决跨站脚本攻击。 ￼![enter image description here](http://drops.javaweb.org/uploads/images/a1c0e64979301404a71ea7ffdb8fe1675f32f139.jpg)

在servlet3.0开始才支持直接通过setHttpOnly设置,其实就算不是JavaEE6也可以在set Cookie的时候加上HttpOnly; 让浏览器知道你的cookie需要以HttpOnly方式管理。而ng a 在新的Servlet当中不只是能够通过手动的去setHttpOnly还可以通过在web.xml当中添加cookie-config(HttpOnly默认开启,注意配置的是web-app_3_0.xsd):

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee 
    http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">
    <session-config>
        <cookie-config>
            <http-only>true</http-only>
            <secure>true</secure>
        </cookie-config>
    </session-config>
    <welcome-file-list>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>

```

还可以设置下session有效期(30分)：

```
<session-timeout>30</session-timeout>

```

### 0x04 CSRF (跨站域请求伪造)

* * *

CSRF（Cross Site Request Forgery, 跨站域请求伪造）用户请求伪造，以受害人的身份构造恶意请求。(经典解析参考：http://www.ibm.com/developerworks/cn/web/1102_niugang_csrf/ )

#### CSRF 攻击的对象

在讨论如何抵御 CSRF 之前，先要明确 CSRF 攻击的对象，也就是要保护的对象。从以上的例子可知，CSRF 攻击是黑客借助受害者的 cookie 骗取服务器的信任，但是黑客并不能拿到 cookie，也看不到 cookie 的内容。另外，对于服务器返回的结果，由于浏览器同源策略的限制，黑客也无法进行解析。因此，黑客无法从返回的结果中得到任何东西，他所能做的就是给服务器发送请求，以执行请求中所描述的命令，在服务器端直接改变数据的值，而非窃取服务器中的数据。所以，我们要保护的对象是那些可以直接产生数据改变的服务，而对于读取数据的服务，则不需要进行 CSRF 的保护。比如银行系统中转账的请求会直接改变账户的金额，会遭到 CSRF 攻击，需要保护。而查询余额是对金额的读取操作，不会改变数据，CSRF 攻击无法解析服务器返回的结果，无需保护。

#### Csrf攻击方式

对象：A：普通用户，B：攻击者

```
1、假设A已经登录过xxx.com并且取得了合法的session，假设用户中心地址为：http://xxx.com/ucenter/index.do
2、B想把A余额转到自己的账户上，但是B不知道A的密码，通过分析转账功能发现xxx.com网站存在CSRF攻击漏洞和XSS漏洞。
3、B通过构建转账链接的URL如：http://xxx.com/ucenter/index.do?action=transfer&money=100000 &toUser=(B的帐号)，因为A已经登录了所以后端在验证身份信息的时候肯定能取得A的信息。B可以通过xss或在其他站点构建这样一个URL诱惑A去点击或触发Xss。一旦A用自己的合法身份去发送一个GET请求后A的100000元人民币就转到B账户去了。当然了在转账支付等操作时这种低级的安全问题一般都很少出现。

```

#### 防御CSRF：

```
验证 HTTP Referer 字段
在请求地址中添加 token 并验证
在 HTTP 头中自定义属性并验证
加验证码
 (copy防御CSRF毫无意义，参考上面给的IBM专题的URL)

```

最常见的做法是加token,Java里面典型的做法是用filter：[https://code.google.com/p/csrf-filter/](https://code.google.com/p/csrf-filter/)(链接由plt提供，源码上面的在：[http://ahack.iteye.com/blog/1900708](http://ahack.iteye.com/blog/1900708))

CSRF的介绍drops已有文章，可以参考：[http://drops.wooyun.org/papers/155](http://drops.wooyun.org/papers/155)
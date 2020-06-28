# Android WebView File域攻击杂谈

0x00 前言
=====

现在越来越多Android App使用了WebView，针对WebView的攻击也多样化。这里简单介绍下目前的WebView File域攻击一些常用的方法，并重点举例说明下厂商修复后依然存在的一些问题。（影响Android 4.4及以下）

0x01 WebView中的file协议
=====

我们知道，在Android JELLY_BEAN以下版本中，如果WebView没有禁止使用file域，并且WebView打开了对JavaScript的支持，我们就能够使用file域进行攻击。

在File域下，能够执行任意的JavaScript代码，同源策略跨域访问能够对私有目录文件进行访问等。APP中如果使用的WebView组件未对file协议头形式的URL做限制，会导致隐私信息泄露。针对IM类软件会导致聊天信息、联系人等等重要信息泄露，针对浏览器类软件，则更多的会是cookie等信息泄露。

譬如之前Wooyun上爆了一个利用file域攻击的漏洞([WooYun: 360手机浏览器缺陷可导致用户敏感数据泄漏](http://www.wooyun.org/bugs/wooyun-2013-037836))

代码如下:

```
var request = false;
    if(window.XMLHttpRequest) {
        request = new XMLHttpRequest();
        if(request.overrideMimeType) {
            request.overrideMimeType('text/xml');
        }
    }
        xmlhttp = request;
    var prefix = "file:////data/data/com.qihoo.browser/databases";
    var postfix = "/webviewCookiesChromium.db"; //取保存cookie的db
    var path = prefix.concat(postfix);
    // 获取本地文件代码 

    xmlhttp.open("GET", path, false);
    xmlhttp.send(null);

```

0x02 使用符号链接同源策略绕过
=====

WebKit 有跨域访问的检查，它的规则是 ajax 访问相同 path 的文件就 allow，否则 deny。利用符号链接，把相同文件名指向了隐私文件，会造成同源策略检查失效。攻击者通过本地文件使用符号链接和File URL结合，利用该漏洞绕过同源策略，进而进行跨站脚本攻击或获得密码和cookie信息。

在JELLY_BEAN及以后的版本中不允许通过File url加载的Javascript读取其他的本地文件，不允许通过File url加载的Javascript可以访问其他的源，包括其他的文件和http,https等其他的源。那么我们要在通过File url中的javascript仍然有方法访问其他的本地文件，即通过符号链接攻击可以达到这一目的，前提是允许File url执行javascript。

目前所知，通过使用符号链接同源策略绕过可以使file域攻击影响到Android 4.4，进一步扩大了攻击范围。

这个典型的漏洞分析可以看下参考栏中的【2】

0x03 依然存在问题
=====

自13年file域导致的安全问题受到关注后，一些厂商进行了相应的修复。因为一些开发者的认识程度以及可能产品实际需求，虽然各种厂商针对性做了不少patch的工作，但是仍然出现了各种问题。当然，这里的前提是WebView 没有禁用file协议也没有禁止调用javascript，即WebView中setAllowFileAccess(true)和setJavaScriptEnabled(true)。我们举几个例子说明如下：

A、某app修复的时候，在某个过程中判断url是否以`file:///`开头，如果是的话就返回，不是的话，再loadUrl

![p1](http://drops.javaweb.org/uploads/images/116bdf63dd0675412923d02a6ee11d6bd03e6cda.jpg)

![p2](http://drops.javaweb.org/uploads/images/779b6108df642154a5009d63d9afc18053370630.jpg)

乍看这个修复，认为已经能够防止file域攻击问题了。但是，忽略了一个问题，如果我们在`file:///`前面添加空格，是否还仍然能够正常loadUrl。尝试后发现会修正下协议头，也就是说file域前面添加空格，是能够正常访问的。于是我们构造了如下POC就能够绕过这个限制：

```
Intent i = new Intent();
i.setClassName("xxx.xxx.xxx","xxx.xxx.xxx.xxxxxx");
i.putExtra("url", " file:///mnt/sdcard/filedomain.html"); //file域前面增加空格
startActivity(i);

```

B、很自然，有些开发者就想trim去空格就可以修复了，于是我们又看到下面这个例子

![p3](http://drops.javaweb.org/uploads/images/ef5a8ef318155c35ae788b7883b62fd19b9fd78d.jpg)

这种修复方式，一般认为已经解决了前面说明的问题了。但是，这里头存在一个技巧，即协议头不包括`///`，还是仍然能够正常loadUrl。看下我们构造了如下POC就能够绕过这个限制：

```
Intent i = new Intent();
i.setClassName("xxx.xxx.xxx","xxx.xxx.xxx.xxxxxx");
i.putExtra("url", "file:mnt/sdcard/filedomain.html"); //file域跟mnt之间没有空格的情况
startActivity(i);

```

通过这两个简单的例子，从细节可以看到虽然现在越来越多的开发者注意到file域攻击的一些问题，也进行了相应的对抗措施，但是仍然还有不少因修复不善导致的绕过的方法，根源还是在于要获取协议头的问题。

0x04 参考
=====

*   【1】[http://blogs.360.cn/360mobile/2014/09/22/webview%E8%B7%A8%E6%BA%90%E6%94%BB%E5%87%BB%E5%88%86%E6%9E%90/](http://blogs.360.cn/360mobile/2014/09/22/webview%E8%B7%A8%E6%BA%90%E6%94%BB%E5%87%BB%E5%88%86%E6%9E%90/)
*   【2】[http://jaq.alibaba.com/blog.htm?spm=0.0.0.0.OK100r&id=62](http://jaq.alibaba.com/blog.htm?spm=0.0.0.0.OK100r&id=62)

0x05 总结
=====

总结一下这类问题的修复

1.  对于不需要使用file协议的应用，禁用file协议
2.  对于需要使用file协议的应用，禁止file协议调用javascript
3.  将不必要导出的组件设置为不导出

File域下攻击的问题，现在已经算是老生常谈了，但是依然还存在很多猥琐的手段来利用。这里不一一说明了，总之思想有多远，攻击面就有多宽！Have Fun！
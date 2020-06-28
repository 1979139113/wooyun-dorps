# Angry Birds和广告系统泄露个人信息——FireEye对Angry Birds的分析

from:[http://www.fireeye.com/blog/technical/mobile-threats/2014/03/a-little-bird-told-me-personal-information-sharing-in-angry-birds-and-its-ad-libraries.html](http://www.fireeye.com/blog/technical/mobile-threats/2014/03/a-little-bird-told-me-personal-information-sharing-in-angry-birds-and-its-ad-libraries.html)

0x00 背景
-------

* * *

很多流行的app，包括愤怒的小鸟在内，收集并且分享玩家的个人信息的广泛程度，远远超过大多数人所了解的。

一些媒体只是进行了表面的报道，没有深入追踪。在去年十月，纽约时报发表了一些关于愤怒的小鸟和其他一些app收集数据的相关报道。在今年2月，多家报社联合公共利益网站ProPublica和英国报社进行一些列报道，详细说明政府机构使用游戏(和一些app)收集用户数据。甚至是历史悠久的CBS在早些时候用了60分钟来对Rovio分享用户位置信息的事情进行了报道。

google应用商店中Android版的愤怒的小鸟是在3月4号更新的，这个版本依旧在分享个人信息。事实上，超过2.5亿的用户为了方便在不同设备上访问，创建了Rovio账户，他们的年龄、性别等等很多信息，会在不知不觉中被分享。并且很多在玩游戏时没有使用Rovio账户的用户，他们的设备信息也会被共享了。

一旦Rovio账户创建，用户信息就会被收集，但是用户很难阻止这些app收集信息。他们把收集到的用户信息可能会放到多个地方:愤怒小鸟的云端，Burstly(广告中介平台)和一些第三方的广告网络，就像Jumptap和Millennial Media这类。虽然用户可以通过在玩愤怒小鸟时不使用Rovio账户来避免信息被收集，但是不能阻止游戏共享这些设备信息。

在这篇blog中，我们将分析“愤怒的小鸟”收集信息的过程。我们还会演示应用程序、广告中介平台和广告云之间的关系，展示三者之间如果共享收集到用户的数据。当然我们会拿出证据来支持我们的观点，比如FireEye Mobile Thread Prevention(MTP)捕获的数据包(PCap)，来支持我们的分析结果。最后我们会通过逆向，在代码层面进行分析。

为了调查信息共享的机制和内容，我们对多个版本进行了研究，发现很多版本都是明文传递收集到的电子邮件，地址，年龄和性别这些信息。

我们发现愤怒的小鸟多个版本存在收集信息的问题，包含最新版本4.1.0(3月4号在google应用商店中更新的版本)，到目前为止，超过2亿用户进行了下载，很多用户会被收集信息。

0x01 什么信息会被收集？
--------------

* * *

Angry Birds鼓励用户创建Rovio账户，这样会给用户带来以下好处:

1、保存分数和游戏中的武器。

2、在不同的设备中保持相同的进度。第二个好处对玩家特别有吸引力，因为它允许在一个设备上停止游戏，一个设备就可以接着上个设备上的进度接着游戏。

图一显示Rovio的注册页面

![enter image description here](http://drops.javaweb.org/uploads/images/f42428710be7a31e6d9c93e75131960a01898a9d.jpg)

图一：Rovio的注册页面

图二显示了在注册过程中收集生日信息，遵循最终使用许可协议(EULA)和授权Rovio可以上传用户信息，同时把信息上传到第三方机构进行营销。

![enter image description here](http://drops.javaweb.org/uploads/images/f27dfba67a78b18af015bfc5cd7350577880e97f.jpg)

图二:Rovio注册过程中收集信息和遵守EULA

图三，注册页面提示用户输入邮件地址和性别。当用户点击注册按钮。Rovio上传收集到的数据到云端，同时创建用户

![enter image description here](http://drops.javaweb.org/uploads/images/c9f3dfd5e1a7b5b1de21ec87c3119b3e1da20085.jpg)

图三:Rovio在注册过程中输入邮箱地址和性别

图四显示愤怒的小鸟通过其他方式收藏用户信息。Rovio推送给注册用户更新游戏的提示，信息片段和特殊优惠。在时事通讯注册过程中，Rovio收集用户的名字、邮箱、生日、居住国和性别。这些信息会和用户的Rovio账户信息的邮箱进行比对

![enter image description here](http://drops.javaweb.org/uploads/images/3850cdbd452a96a0bb775f9aece77e58f07a5e94.jpg)

图四:时事通讯注册页面要求更多的用户信息

![enter image description here](http://drops.javaweb.org/uploads/images/2cd284b86f79df366a62b3001ff1ad0fb3e91b34.jpg)

图五:愤怒的小鸟、广告平台和广告云之间的信息流。

首先，我们关注的是那些信息被传送到广告库中。图5显示了“愤怒小鸟”的信息流：“愤怒小鸟”的云端，Burstly(内嵌广告库和广告中介平台)，以及Jumptap和Millennial Media等这些基于云的广告服务。

“愤怒的小鸟”通过Burstly对广告进行植入，它提供了大量的第三方广告云包括Jumptap和Millennial Media，通过识别用户信息来实现精确广告推送。

如图所示。愤怒的小鸟保持与广告平台Burstly和广告提供商Millennial Media之类相关平台的HTTP连接。

数据流： 表一是一个总结，下面会有详细讲解

![enter image description here](http://drops.javaweb.org/uploads/images/f76a64041cf0700d70aa88d575804b6276f5c33a.jpg)

表1：“愤怒的小鸟”，Burstly和第三方广告云之间的信息传递

“愤怒的小鸟”在本地调用叫做libAngryBird.so的代码来访问存储，同时帮助广告库存储日志，缓存，数据库，配置文件和AES加密游戏数据。对于用户使用的Rovio账户，这些包含用户信息的数据是明文或者只是进行了简单的加密。

例如:一些信息被明文存储在缓存中，被称为webviewCacheChromium。

```
{“accountId”:”AC3XXX…XXXA62B”,”accountExtRef”:”hE…fDc”,”personal”:{“firstName”:null,”lastName”:null,“birthday”:”19XXXXX-01″, “age”:”30″, “gender”:”FEMALE”, “country”:”United States” , “countryCode”:”US”, “marketingConsent”:false, “avatarId”:”AVXXX…XXX2c”,”imageAssets”:[...], “nickName”:null}, “abid”:{“email”:”eXXX…XXXe@XXX.XXX”, “isConfirmed”:false}, “phoneNumber”:null, “facebook”:{“facebookId”:”",”email”:”"},”socialNetworks”:[]}

```

这个设备被赋予一个通用id 1XXXXX8,这是明文存储在webviewCookieChromiun

```
cu1XXXX8|{“name”:”cu1XXXX8“,”value”:”3%2XXX…XXX6+PM”}|13XXX…XXX1

```

当上传用户信息到广告中介平台时，这个id "1XXXX8"是用户信息的标签。然后将信息传递给广告云

1、捕获的初始数据包流量显示什么信息会被"愤怒的小鸟"上传到Burstly

```
HTTP/1.1 200 OK
Cache-Control: private
Date: Thu, 06 Mar 2014 XX:XX:XX GMT
Server: Microsoft-IIS/7.5
ServerName: P-ADS-OR-WEBC #22
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
X-ReqTime: 0
Content-Length: 0
Connection: keep-alive

POST /Services/PubAd.svc/GetSingleAdPlacement HTTP/1.1
Content-type: text/json; charset=utf-8  
User-Agent: Mozilla/5.0 (Linux; U; Android 4.4.2; en-us; Ascend Y300 Build/KOT49H) AppleWebKit/534.30 (KHTML, like Gecko) Version/4.0 Mobile Safari/534.30

Content-Length: 1690
Host: neptune.appads.com
Connection: Keep-Alive

{“data”:{“Id”:”8XXX5″,”acceptLanguage”:”en”,”adPool”:0,”androidId”:”u1XXX…XXXug”,”bundleId”: “com.rovio.angrybirds”,…,”cookie”:[{"name":"cu1XXX8","value":"3XXX6+PM"},{"name":"vw","value":"ref=1XXX2&dgi=,eL,default,GFW"},{"name":"lc","value":"1XXX8"},{"name":"iuXXXg","value":"x"},{"name":"cuXXX8","value":"3%2XXXPM"},{"name":"fXXXg","value":"ref=1XXX712&crXXX8=2,1&crXXX8=,1"}], “crParms”:”age=30,androidstore=’com.android.vending’, customer=’googleplay’, gender=’FEMALE’, version=’4.1.0′”, “debugFlags”:0, “deviceId”:”aXXX…XXXd”, “encDevId”:”xXXX….XXXs=”, “encMAC”:”iXXX…XXXg=”, “ipAddress”:”",“mac”:”1XXX…XXX9″, “noTrack”:0,”placement”:”", “pubTargeting”:”age=30, androidstore=’com.android.vending’, customer=’googleplay’, gender=’FEMALE’, version=’4.1.0′”,”rvCR”:”", “type”:”iq”,”userAgentInfo”:{“Build”:”1.35.0.50370″, “BuildID”:”323″, “Carrier”:”",”Density”:”High”, “Device”:“AscendY300″, “DeviceFamily”:“Huawei”, “MCC”:”0″,”MNC”:”0″,…

```

我们可以看到上传到neptune.appads.com的信息包括，性别、年龄、安卓id、设备id、mac地址、设备类型等等

在其他数据包中看到、"愤怒的小鸟"通过POST把IP地址发送到同一个主机

```
HTTP/1.1 200 OK
…
POST /Services/v1/SdkConfiguration/Get HTTP/1.1
…
Host: neptune.appads.com
…
IpAddress”:”fXXX…XXX9%eth0″,…

```

根据Whois记录，注册neptune.appads.com的组织就是Burstly,上述信息被发送到Burstly。在数据包中都包含 “crParms.” 这个关键字，当源码发送个人信息时，这个关键字是作为有效载荷的。

Skyrocket.com是Burstly提供的一个应用货币虚拟化的服务。跟踪数据包，发现"愤怒的小鸟"，通过HTTP GET请求，从Skyrocket.com查询用户ID

```
HTTP/1.1 200 OK
Cache-Control: private
Content-Type: text/html
Date: Thu, 06 Mar 2014 07:12:25 GMT
Server: Microsoft-IIS/7.5
ServerName: P-ADS-OR-WEBA #5
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
X-ReqTime: 2
X-Stats: geo-0
Content-Length: 9606
Connection: keep-alive

GET /7….4/ad/image/1…c.jpg HTTP/1.1
User-Agent: Mozilla/5.0 (Linux; U; Android 4.4.2; en-us; Ascend Y300 Build/KOT49H) AppleWebKit/534.30 (KHTML, like Gecko) Version/4.0 Mobile Safari/534.30
Host: cdn.skyrocketapp.com
Connection: Keep-Alive

{“type”:”ip”,”Id”:”9XXX8″,…”data”:[{"imageUrl":"http://cdn.skyrocketapp.com/79...2c.jpg","adType":{"width":300, "height":250, "extendedProperty":80}, "dataType": 64, "textAdType":0,"destType":1,"destParms":"","cookie":[{"name":"fXXXg", "value": "ref=1XXX2&cr1XXX8=2,1&cr1XXX8=1&aoXXX8=", "path":"/", "domain": "neptune.appads.com", "expires":"Sat, 05 Apr 2014 XXX GMT", "maxage": 2…0}, {"name":"vw","value":"ref=1XXX2&...},...,"cbi":"http://bs.serving-sys.com/Burstin...25&rtu=-1","cbia":["http://bs….":1,"expires":60},..."color":{"bg":"0…0"}, "isInterstitial":1}

```

2、在这个数据包中。通过HTTP POST从jumptap.com(即Millennial Media)获得包含用户id的广告信息

```
HTTP/1.1 200 OK
Cache-Control: private
Content-Type: text/html
Date: Thu, XX Mar 2014 XX:XX:XX GMT
Server: Microsoft-IIS/7.5
ServerName: P-ADS-OR-WEBC #17
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
X-ReqTime: 475
X-Stats: geo-0;rcf88626-255;rcf75152-218
Content-Length: 2537
Connection: keep-alive

GET /img/1547/1XXX2.jpg HTTP/1.1
Host: i.jumptap.com
Connection: keep-alive
Referer: http://bar/
X-Requested-With: com.rovio.angrybirds
User-Agent: Mozilla/5.0 (Linux; U; Android 4.4.2; en-us; Ascend Y300 Build/KOT49H) AppleWebKit/534.30 (KHTML, like Gecko) Version/4.0 Mobile Safari/534.30
Accept-Encoding: gzip,deflate
Accept-Language: en-US
Accept-Charset: utf-8, iso-8859-1, utf-16, *;q=0.7

{"type":"ip","Id":"8XXX5","width":320,"height":50,"cookie":[],”data”:[{"data":"<!-- AdPlacement : banner_ingame_burstly…","adType":{"width":320, "height":50, "extendedProperty":2064 },"dataType":1, "textAdType":0, "destType":10, "destParms":"", "cookie":[{"name":"...", "value":"ref=...&cr1XXX8=4,1&cr1XXX8=2,1", "path":"/", "domain":"neptune.appads.com", "expires":"Sat, 0X Apr 2014 0X:XX:XX GMT", "maxage":2XXX0}, {"name":"vw",..., "crid":7XXX2, "aoid":3XXX3, "iTrkData":"...", "clkData":"...","feedName":"Nexage"}]}

```

这个数据包中，广告是从jumptap.com得到，我们也能使用相同的用户id"1XXXX8"去跟踪不同的广告库

3、例如，在其他来自turn.com的数据包，用户id和刚才的相同

```
HTTP/1.1 200 OK
Cache-Control: private
Content-Type: text/html
Date: Thu, 06 Mar 2014 07:30:54 GMT
Server: Microsoft-IIS/7.5
ServerName: P-ADS-OR-WEBB #6
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
X-ReqTime: 273
X-Stats: geo-0;rcf88626-272
Content-Length: 4714
Connection: keep-alive

GET /server/ads.js?pub=24…
PvctPFq&acp=0.51 HTTP/1.1
Host: ad.turn.com
Connection: keep-alive
Referer: http://bar/
Accept: */*
X-Requested-With: com.rovio.angrybirds
User-Agent: Mozilla/5.0 (Linux; U; Android 4.4.2; en-us; Ascend Y300 Build/KOT49H) AppleWebKit/534.30 (KHTML, like Gecko) Version/4.0 Mobile Safari/534.30
Accept-Encoding: gzip,deflate
Accept-Language: en-US
Accept-Charset: utf-8, iso-8859-1, utf-16, *;q=0.7

{“type”:”ip”,”Id”:”0…b”,”width”:320,”height”:50,”cookie”:[],”data”:[{"data":"<!-- AdPlacement : banner_ingame_burstly --> "http://burstly.ads.nexage.com:80..." destParms":"", "cookie":[{"name":"f...g", "value":"ref=1...0&cr1XXXX8=k,1&cr...8=i, 1","path":"/", "domain":"neptune.appads.com", "expires":"Sat, 0X Apr 2014 0X:XX:XX

```

0x02 个人信息是怎么被分享的？
-----------------

* * *

我们研究了Burstly(广告中介平台)的源代码，来跟踪叫做信息贡献的方法。

首先在`com/burstly/lib/conveniencelayer/BurstlyAnimated Banner.java`

当"愤怒的小鸟"和Burstly初始话连接时，initNewAnimatedBanner() 会被调用

```
this.initNewAnimatedBanner (arg7.getActivity(), arg8, arg9, arg10, arg11);
Inside initNewAnimatedBanner(), it instantiates the BurstlyView object by calling:
BurstlyView v0 = new BurstlyView(((Context)arg3));
v0.setZoneId(arg6);

```

在Zoneld设置之前， initializeView()被BurstlyView这个构造函数调用，此外在 initializeView() 中，我们发现以下几点：

```
new BurstlyViewConfigurator(this).configure(this.mAttributes);

```

最后在 BurstlyViewConfigurator.configure()中，设置了一些参数

```
this.extractAndApplyBurstlyViewId();
this.extractAndApplyCrParams();
this.extractAndApplyDefaultSessionLife();
this.extractAndApplyPublisherId();
this.extractAndApplyPubTargetingParams();
this.extractAndApplyUseCachedResponse();
this.extractAndApplyZoneId();

```

下面这个方法被用作从burstly.com还原信息。例如，`在extractAndApplyCrParams()`中,从burstly.com得到一些参数同时把它们存储到BurstlyView对象

```
String v0 = this.mAttributes.getAttributeValue("http://burstly.com/lib/ui/schema", "crParams");
if(v0 != null) {
BurstlyViewConfigurator.LOG.logDebug("BurstlyViewConfigurator", "Setting CR params to: {0}", new Object[]{v0});
this.mBurstlyView.setCrParms(v0);
}

```

关键字crParms和第一个数据包中的标签一样，都是被用来标识像年龄和性别这类的用户信息

0x03 结论
-------

* * *

综上所述、"愤怒的小鸟"会收集用户个人信息和自定义的ID，在存储到本地之前进行上传。然后Burstly的广告库导入到“愤怒的小鸟”的自定义ID，上传相应的个人信息给Burstly云和其他广告云系统。

我们在网络中捕获到响应的数据包、同时逆向分析了代码。
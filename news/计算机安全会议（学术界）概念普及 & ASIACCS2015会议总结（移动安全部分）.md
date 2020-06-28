# 计算机安全会议（学术界）概念普及 & ASIACCS2015会议总结（移动安全部分）

0x00 序
=====

ASIACCS 2015 全称为10th ACM Symposium on Information, Computer and Communications Security。因为ASIACCS在ACM还有个兄弟会议叫CCS (ACM Conference on Computer and Communications Security), 又因为会议举办地点几乎都在亚洲，于是缩写就变成了ASIACCS。今年的会议一共有269篇paper投稿，48篇被accepted，中稿率为18%。

在讨论会议内容前，我先简单科普一下计算机安全领域会议的一些基本概念（学术界大牛可以直接跳过看下一章）。

0x01 计算机安全会议（学术界）概念普及
=====

1． 每个会议发出论文邀请以及会议的主页，并列出了投稿主题和兴趣，以及截止投稿时间。每个学术会议一般一年举办一次，最有名的学术界四大安全会议为S&P（又称Oakland），Usenix Security，CCS和NDSS (排名分先后)。

2．研究员在截止日期之前提交论文。每个会议通常会收到100至300篇论文投稿，语言必须为英文，每篇论文包含等同于8-15页双栏或者20-40页单栏的文字量（包括引用）。有的会议还接受short paper（短篇论文），比full paper（完整论文）大概少一半的文字量。

3．Program Committee（会议程序委员会） 由大约20 - 50位专家学者组成。每篇文章会被三到五个人评审，这些人要么是PC成员，要么是由PC成员邀请的志愿外部审稿专家。审稿过程会持续约1-3个月左右。

4．在所有PC成员完成审稿后，PC会参考审稿人的建议开会决定接受哪些论文和拒收哪些论文。比较好的会议的中稿率都会在20%以下，二流会议的中稿率大概在20%-30%之间。随后PC向所有作者发email通知他们是否接受其论文，并附上审稿意见。因为这些相对较低的接受率，一篇论文在被接受发表之前被拒收，修改，重新提交数次是不奇怪的，这个过程可能会花费数年。（一篇文章在同一时间只能被提交到一个会议。）

5．论文被接受的作者一般会要求在一个月之内提交Camera ready（终稿）。随后大概在结果出来之后的2-3个月左右去参加会议，并做一个关于其论文的30分钟的报告。所有被接受的文章都会被在线收录到数码图书馆（比如ACM library和IEEE library）。

6．会议举行的目的不光是为了作者们的演讲报告，也是一个学术界进行social（社交）的主要形式。除了作者们的报告，PC还会邀请业界的名人做keynote speech（专题演讲），并举行晚宴。在会议中你可以见到其他大牛和同行，并和他们进行交流。并且会议的举办地点会在世界各地，所以也是一个出国旅游的好机会。

7．如果还有兴趣继续了解计算机方面的会议和PhD的生活，可以去读一本叫The Ph.D. Grind的书。 下载地址：http://pgbovine.net/PhD-memoir.htm

0x02 ASIACCS2015总结（移动安全部分）
=====

本次ASIACCS一共收取了5篇移动安全方向的论文，我会一一进行讲解。

1 Towards Discovering and Understanding the Unexpected Hazards in Tailoring Antivirus Software for Android
----------------------------------------------------------------------------------------------------------

* * *

作者首先提出安卓上的杀毒引擎AVDs(Android Virus Detectors )有着很高的查杀率(95%)，并且他认为拥有高查杀率的原因是杀毒厂商的病毒库非常完善。但是他发现杀毒软件的扫描并不是实时的，并且现在有大量的malware采用了动态加载技术，因此，如果杀毒软件不能在病毒加载恶意payload的时候进行扫描，它是无法获取到病毒特征并查杀到这种采用动态加载技术的病毒的。为了论证这一点，作者做了大量的实验来测试在google play下载量最高的30款杀毒软件，并发现了很多bypass扫描的方法。他把各种bypass的方法称为Hazard。最有趣的Hazard是：作者首先构造一个复杂的zip tree文件（也就是zip里面包含zip，这样连续套好几层的大zip文件）放在sdcard下面，然后观察CPU的状态，作者发现这个zip tree文件可以让很多杀毒软件delay很长时间，甚至可以对某些杀毒软件进行DOS (denial of service)攻击。另外作者还发现，在杀毒软件升级的时候会出现null-protection window (无防护时间窗口)，病毒可以通过监听系统广播（比如Package_REMOVED，Package_REPLACED等）来找到合适的null-protection window，随后动态释放payload，执行恶意代码。文章随后作者给出了一些意见给杀毒厂商以及Google。

（此段非论文内容）关于系统广播这一点，我去年在美国实习的时候发现了一个非常有趣的现象，不知道算不算bug：1. 新安装的app是处在一个stopped state的，在这个state下的app是无法接收任何广播的 (1]，但是如果是一个updated app，并且original app不在stopped state，这个updated app是可以接收广播的。2. 新安装的app，包括updated app是无法接收自己的PACKAGE_ADDED这个广播的 (2]，但是在安装过程中，updated app却可以收到original app的PACKAGE_REMOVED广播！打个比方，一个人的还在出生的过程中，却可以听到前世死掉时的声音，真的是非常诡异。这个机制可以用来干什么呢？好的方面，他可以解决杀毒软件的无防护时间窗口问题，当杀毒软件update后可以通过接收PACKAGE_REMOVED广播立刻启动杀毒应用，而不用等待用户去手动启动杀毒软件或者重启手机。坏的方面病毒也可以配合master key漏洞，做到替换某个app后立刻执行恶意行为而不用等待用户去手动点击app或者重启手机。该bug在4.4版本以及之前版本都测试通过，并写信通知了Google，但不知道Google是否打算修复这个问题。

2 Hybrid User-level Sandboxing of Third-party Android Apps
----------------------------------------------------------

* * *

作者首先介绍android上的app经常会获取过多的权限，并且在用户无意识的情况下泄漏个人隐私信息，并且现在的恶意app很多都是在native层实现恶意行为，传统的针对dalvik层的hook防护机制已经不那么有效了。因此作者提出了AppCage这个系统。这个系统可以对原app进行重打包处理，然后在android上实现dalvik层和native层的双重sandbox机制。首先讲解的是dalvik层的sandbox，实现方法是hook dalvik虚拟机。因为dvm会在内存中维护一个ClassObject的数据结构，通过这个数据结构可以找所有方法的引用。因此AppCage会hook一些比较危险的方法（比如发送短信的方法），如果一个App调用了这些方法，AppCage会弹出对话框来询问用户是否同意执行，如果同意了，AppCage才会调用原方法执行。接下来作者介绍了native层的sandbox。首先我们知道在native层动态修改dex层的code已经是一种很成熟的技术了，通过native层的sandbox可以有效的防止native层的code篡改内存中dex的code。其次，native层的sandbox还可以监测和阻止危险的指令比如system call（系统调用）。对于app本身的native code，AppCage具体的实现方法是采用binary rewrite的方式，因为假设无法获取原app的源码，所以需要先反编译原native library，随后对代码进行插桩来保证native code无法修改或执行除了sandbox分配的内存空间以外的数据，同时还可以监控system call的调用。针对android系统的library，AppCage采用了NaCl compiler sandbox了对应的系统library（比如libc.so），并且让app在运行前加载sandbox过后的system library。针对JNI，AppCage采用hook dlopen和dlsym方法来解决dalvik和native的context不同的问题。在系统实验部分，作者下载了google play前50的app以及一些malware进行了测试，实验证明AppCage不光成功的阻止了某些app的危险行为，并且重打包后的AppCage app无论是在运行效率方面还是在文件大小方面都处在可接受范围之内。

这篇paper的一作是Yajin Zhou (3]，他来自北卡州立大学的Xuxian Jiang教授的团队。如果你经常看android安全方面的论文，那你一定看过Jiang教授团队的paper，他们的论文无论是引用数还是质量方面都是世界上最顶尖的。

3 On Designing an Efficient Distributed Black-Box Fuzzing System for Mobile Devices
-----------------------------------------------------------------------------------

* * *

这篇paper主要讲述了如何在android和iOS上进行黑盒fuzzing来找漏洞。作者提出了一个系统叫MVDP (Mobile Vulnerability Discovery Pipeline)，这个系统可以用来生成图片，音频和视频文件，然后分发到相应的设备上（android/iOS），随后收集崩溃信息(crash report)。作者首先提出，想要做好fuzzing最重要就是要有好的种子文件(seed file)，作者采用的方式是先用搜索引擎在网上下载各种候选种子文件，然后使用SFAT（种子文件分析选择器）去分析种子文件的数据结构，如果这个种子文件包含的数据越丰富分数越高，越会被选中。选好seed file之后，就会进入种子模糊环节，在这个环节中系统的Fuzzing Engine会对种子文件进行变异处理，比如删除，增加或修改一些种子文件中的数据，随后会使用评估工具对模糊后的种子文件进行打分，越具有独特性的种子文件评分越高。接下来，MVDP系统会将种子文件传送到目标机器上并打开，然后收集crash report，最后再分析crash report来发现漏洞。作者的实验一共持续了2周，大概触发了1900个crash report并发现了7个漏洞，不过这些漏洞还没有严重到发CVE的程度。

4 XiOS: Extended Application Sandboxing on iOS
----------------------------------------------

* * *

这篇paper主要介绍了一种新的sandbox iOS app的方法，用来防护比较流行的iOS上的攻击手段（比如Usenix Security上提出Jekyll app）。作者首先介绍了几种比较流行的iOS攻击手段，比如iOS app采用动态加载调用private API，或者故意上传有漏洞的app到App Store，然后采用ROP的方式触发并利用漏洞等。为了防止这类攻击，作者提出了XiOS这个系统，这个系统采用binary rewrite的方式，对已经编译好后的iOS app加入sandbox，然后重打包后的XiOS app可以上传到App Store或者采用企业证书发布。XiOS的防护手段主要是采用instrumentation的方式hook了动态加载的API调用，随后所有的动态API调用都会经过XiOS的Reference Monitor检测，如果这个API在允许列表里才会执行。同时XiOS还加入了一些防护机制用来防止app篡改Shadow Table，也就是XiOS用来保存数据的区域。

这篇paper思路几乎和AppCage如出一辙，只是AppCage是android上的sandbox，XiOS是iOS上的sandbox。还有个不同点是AppCage是让用户在app运行中进行选择，XiOS是基于自定义规则进行防护的。但是因为这两种sandbox都是在用户层实现的，如果针对他们的hook做防护的话应该是可以bypass的。

5 Enpublic Apps: Security Threats Using iOS Enterprise and Developer Certificates
---------------------------------------------------------------------------------

* * *

这篇paper主要介绍了iOS上一种最近非常流行的安全威胁：利用开发者证书或者企业证书来分发iOS app。作者刚开始提到iOS上的病毒比Android上要少很多，并认为这主要是App Store严格审核的功劳。虽然有些论文提出了bypass审核的方法，但施行起来都比较复杂。随后作者提出了一种非常简单的bypass的方式，也就是利用开发者证书或者企业证书来分发iOS app。虽然苹果不允许企业app被非工作人员使用，但是有很多公司的确在利用企业证书分发app，作者把这类app称为enpublic app。Enpublic app大多都是通过itms协议来安装的，作者利用搜索引擎可以在网上找到大量的enpublic app，通过分析app的归属地发现这些app大多来自美国和中国。随后作者讲述了如果获取private API列表，然后通过列表判断enpublic app是否使用了private API。接下来作者还对enpublic app的URL scheme进行fuzzy测试，来测试是否有URL漏洞。作者最后总结出了25个危险的private API调用，并且发现绝大多数的enpublic app都会调用private API。另外作者还发现了2个zero-day的漏洞并提交给了apple，apple在7.1修复了用户监控漏洞（CVE-2014-1276），并打算在未来版本中修复钓鱼漏洞。PS：这篇paper中提到的钓鱼漏洞和URL漏洞可以参考(4]和(5]这两篇文章。

0x03 花絮
=====

今年ASIACCS的举办地是在新加坡，是一个非常热非常小的国家（接近赤道）。会议一共是4天，第一天报道和workshop，随后三天开主会。新加坡最有特色的地方就是鱼尾狮和金沙酒店上面的露天游泳池了。最好吃的东西当然要属辣椒螃蟹和肉骨茶。如果去新加坡旅游千万不要错过。最后附几张照片。

![enter image description here](http://drops.javaweb.org/uploads/images/d83e76726cacdb5855a562e8ea88c0fdc91d3b34.jpg)

0x04 总结
=====

从今年的ASIACCS论文（移动安全方向）来看，iOS已经开始变成一个热门topic了，因为iOS的paper还非常少，有很多坑可以挖，在未来2年一定是个大热。再看android方面，malware相关的paper已经越来越少，漏洞挖掘以及完善AOSP系统开始变为主流。WP方面依然冷清，几乎没人研究。最后附上5篇移动安全的论文下载地址(6]，有兴趣的同学可以继续研究。

0x05 参考文献
=====

1.  stopped state http://developer.android.com/about/versions/android-3.1.html
    
2.  PACKAGE_ADDED Intent http://developer.android.com/reference/android/content/Intent.html#ACTION_PACKAGE_ADDED
    
3.  Yajin Zhou https://scholar.google.com/citations?user=N1oeOPwAAAAJ
    

4．在非越狱的 iPhone 6 (iOS 8.1.3) 上进行钓鱼攻击 (盗取 App Store 密码) http://blog.csdn.net/zhengminwudi/article/details/43916791

5．iOS URL Scheme 劫持-在未越狱的 iPhone 6上盗取支付宝和微信支付的帐号密码http://drops.wooyun.org/papers/5309

6．论文打包下载 http://www.cse.cuhk.edu.hk/~mzheng/paper/asiaccs2015.zip
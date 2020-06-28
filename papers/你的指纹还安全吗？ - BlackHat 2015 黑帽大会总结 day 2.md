# 你的指纹还安全吗？ - BlackHat 2015 黑帽大会总结 day 2

0x00 序
=====

今天是Black Hat 2015第二天，第一天的大会总结请参考：

看黑客如何远程黑掉一辆汽车 -[BlackHat 2015 黑帽大会总结 day 1](http://drops.wooyun.org/papers/7716)

0x01 TRUSTKIT: CODE INJECTION ON IOS 8 FOR THE GREATER GOOD
=====

本来打算去听shendi的TrustZone crack的talk，但是因为shendi的visa没有办下来，最后就给cancel了。于是去听了这个iOS injection的talk。

Talk首先介绍说在iOS 8之前是不允许动态加载library的，只允许静态编译。但在iOS 8之后加入了新的特性叫”Embedded Frameworks”。允许应用和它的extension进行通讯，但实际上无论有没有extension都可以用这个新特性。

然后作者尝试用substrate的方法来hook SSL方法在非越狱机器上，但是失败了，原因是substrate在hook函数的时候需要patch函数的prologue，因此需要RWX权限，但这在非越狱机器上是不可能的。最终作者采用的方法是facebook开发的fishhook框架用来进行HOOK。`https://github.com/facebook/fishhook`

最后作者利用上面讲到的方法开发了trustkit。利用trustkit，可以在不修改源代码的情况下用来hook SSL函数并加入pinning的功能。并且作者在今天发布了源码。

PPT下载：

```
https://www.blackhat.com/docs/us-15/materials/us-15-Diquet-TrustKit-Code-Injection-On-iOS-8-For-The-Greater-Good.pdf

```

trustkit源码：

```
https://github.com/datatheorem/TrustKit

```

0x02 AH! UNIVERSAL ANDROID ROOTING IS BACK
=====

![enter code here](http://drops.javaweb.org/uploads/images/320161d868c51bc8fd753f4d53e62981fcfb0303.jpg)

speaker是 Xu Wen，来自上海交通大学并且在keen实习。首先speaker提到想要实现万能root必须要基于linux kernel的漏洞而不是靠驱动的漏洞。随后speaker讲述了怎么样发现漏洞的过程。

speaker首先采用`https://github.com/kernelslacker/trinity`这个system call fuzzer获取到很多的log信息。然后在分析fuzzer的log的过程中，发现kernel crash在一个很奇怪的地址0x200200。继续分析发现问题是由ping_unash()这个函数引起的。但开始只能造成拒绝服务攻击，这对于root来说是远远不够的。随后继续分析发现sock_put(sk)被调用了2遍，会产生非常普遍的UAF（use-after-free）漏洞。既然有了UAF漏洞，下一步就是用UAF来控制内核。PingPong Root采用的方式是Ret2dir (Ret2dir: rethinking kernel isolate)。控制内核以后就是执行提权的shellcode了。PingPong root借鉴了towelroot中使用的通用提权shellcode，让进程获取root权限。

speaker最后提到了如何root 64位的设备。在64位上，UAF漏洞依然存在，但有一个问题是无法将shellcode return到user space。所以需要做kernel层的ROP。最后采用了类似rop的JOP（jump to program）的方法，并在某些ROMs中发现了一个GOD gadgets可以做到泄露内存信息以及覆盖数据的功能。最终做到了root提权。

PPT：

```
https://www.blackhat.com/docs/us-15/materials/us-15-Xu-Ah-Universal-Android-Rooting-Is-Back.pdf

```

0x03 FINGERPRINTS ON MOBILE DEVICES: ABUSING AND LEAKING
=====

speaker是来自FireEye的Wei Tao和Zhang Yulong。Talk首先介绍了指纹系统的原理以及实现，比如如何进行特征采集，如何对比特征等。

![enter image description here](http://drops.javaweb.org/uploads/images/5d37606960f0cd55b28884fb1c9c8d2cb065edb9.jpg)

随后讲了2种架构，一种是Fingerprint without TrustZone和Fingerprint with TrustZone。在root情况下without TrustZone是非常危险的，所有的数据都可以轻松获取到。但是在有了TrustZone的情况下，hacker在获取了root以后依然无法读取TrustZone中的指纹信息。如果想要获取指纹信息，理论上还需要破解TrustZone才行。

![enter image description here](http://drops.javaweb.org/uploads/images/a144dc210c14d69d6452994da915362a240ada1f.jpg)

接着speaker介绍了四种攻击手段：第一个攻击是confused attack（迷惑性攻击）。speaker提到fingerprint有两种用处，一种是authentication另一种authorization。就像是passport和visa。一个用来做身份验证，一个用来行使权力。Hacker可以采取一种迷惑性的攻击，让用户仅仅是觉得做身份验证，但实际上却行使了权利，比如说在demo中用户以为他在解锁手机屏幕，而实际上却使用指纹转了钱给黑客。

![enter image description here](http://drops.javaweb.org/uploads/images/e63bbe6065f8517e41cfc05c2a4619d625fce9f8.jpg)

第二个攻击是不安全的数据存储。最经典的例子就是HTC one的指纹保存文件。对所有人都是可读可写的。毫无安全性可言。

![enter image description here](http://drops.javaweb.org/uploads/images/89270c73287bd029f698f1b6f21b4b3b088039b0.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/36960f6c5043d54a0fbc5299242f7c21543da1c7.jpg)

第三个攻击是finger spy。虽然TrustZone非常安全，但是android系统是通过应用层的service和TrustZone进行通讯的。因此hacker可以伪造一个finger print app，并且和finger print sensor进行通讯，从而窃取到用户的指纹。三星针对这个问题的解决方式是TrustZone UI。也就是当使用指纹进行授权的时候必须通过TrustZone UI来进行，因为TrustZone UI也是TrustZone的一部分，所以黑客必须要破解掉TrustZone才能获取到指纹。

![enter image description here](http://drops.javaweb.org/uploads/images/27ce1af6818d02befa5efff1b951eebd6059d2e5.jpg)

第四个攻击是fingerprint backdoor。用户在系统的设置中可以查看当前记录的指纹数量，但是这个数量信息并没有保存在TrustZone当中。因此hacker可以留下自己的指纹作为后门，并且将增加的指纹的数量减掉。比如说在demo中fingerprint service显示仅保存了一个指纹，但是demo中却成功的用三个指纹解锁了手机屏幕，因为其中两个指纹其实是黑客留下的，为了防止用户发现，黑客将保存的指纹数修改成了1。

PPT：

```
https://www.blackhat.com/docs/us-15/materials/us-15-Zhang-Fingerprints-On-Mobile-Devices-Abusing-And-Leaking.pdf

```

0x04 REVIEW AND EXPLOIT NEGLECTED ATTACK SURFACES IN IOS 8
=====

这个talk来自盘古。speaker首先介绍了iOS的几个攻击点：本地攻击，远程攻击，内核攻击等，并举了很多例子（比如之前jailbroken使用的漏洞）。

接着speaker介绍了在kernel层进行fuzzing的tips。speaker首先提到IOKit是最好的fuzz目标，因此对fuzzy IOkit的第一个建议是尽量fuzz更底层的函数。比如说IOConnectCallMethod里有对参数大小的限制，但是如果调用io_connect_method就没有参数大小的限制。第二个建议就是使用information leak的漏洞来获取fuzzy过程中产生的信息。

随后speaker提到了Shared Memory。因为IOKit会share一些data到用户层并且用户层可以对这些数据进行修改。因此在fuzz io_connect_method的过程中可以对这些用户层数据也进行fuzz。随后IOKit在读取了用户层修改的数据之后就有可能产生漏洞。接着speaker又介绍了iokit_user_client_trap()这个用户层函数的fuzzy，并demo了一个0day。

接下来speaker介绍了如何在用户层挖XPC的洞。首先在应用上分别建立server端和client端，然后使用client端对server端进行通讯。

![enter image description here](http://drops.javaweb.org/uploads/images/8557fe32229b5969ba0bec82b91d97c0a54ed2c1.jpg)

在进行通讯过程的中，有很多函数可以进行fuzz，如果函数对传输的数据处理有误，有可能会产生空指针异常，内存越界以及远程代码执行等漏洞。最后演讲者分别展示这几种漏洞的POC代码并简单介绍了如何利用这些fuzz出来的漏洞。

PPT：

```
https://www.blackhat.com/docs/us-15/materials/us-15-Wang-Review-And-Exploit-Neglected-Attack-Surface-In-iOS-8.pdf

```

0x05 总结
=====

最后一个talk本来想去听360讲的Android Fuzzing，结果又是因为签证问题给cancel了。于是本届BlackHat的会议部分就算结束了。明天开始将在隔壁酒店举行DEFCON的会议以及CTF比赛。欢迎大家继续关注。
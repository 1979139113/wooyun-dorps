# OS X平台的Dylib劫持技术（下）

原文：[https://www.virusbtn.com/virusbulletin/archive/2015/03/vb201503-dylib-hijacking](https://www.virusbtn.com/virusbulletin/archive/2015/03/vb201503-dylib-hijacking)

0x01 攻击
=====

跟着我们对OS X平台下的dylib劫持知识有了基础的理解以后，现在是时候来看看现实世界中的攻击场景，并且提供一些实际的防御方法。

高级的黑客知道，尽可能多的攻击组建自动化对攻击的重要性。这样的自动化能提升攻击的规模和效率。解放攻击者，让攻击者将精力集中到更高的要求或者攻击中更复杂的方面。

劫持攻击的第一个自动化组件就是程序漏洞的挖掘。我们用一个Python脚本，dylibHijackScanner.py(下载链接【15】),用来完成此项任务。该脚本在收集运行进程列表和系统上所有可执行程序后，智能分析二进制文件的Mach-O头结构和加载命令。为了挖掘可能通过弱dylib加载命令而被劫持的二进制文件，该脚本试图寻找指向不存在dylib文件的LC_LOAD_WEAK_DYLIB加载命令。

[[email protected]](http://drops.com:8000/cdn-cgi/l/email-protection)制文件就有些复杂了。首先，该脚本寻找那些至少包含一个LC_LOAD*_DYLIB加载命令指向run-path依赖库的二进制文件。如果这样的加载命令被发现，脚本就继续分析二进制文件的加载命令，寻找多个LC_RPATH。如果这两个先决条件都满足，脚本继续检查是否在第一个run-path搜索路径中找到run-path-denpendent library。如果该库不存在，脚本就会提示用户这个程序存在漏洞。执行该程序发现了大量的漏洞程序，当然包括我们的测试程序rPathApp.app。

![](http://drops.javaweb.org/uploads/images/8c750378e6524136e3a736352430e869a5ca9e6f.jpg)

图33 自动挖掘漏洞程序

我们可以在图33中看到，扫描脚本在作者的电脑上找到了近150个漏洞二进制文件。有趣的是，大部分漏洞程序都属于更加复杂的‘multiple rpath’一类。由于空间局限，完整的漏洞程序列表无法在这里呈现。但是表1列出了几个广泛使用的知名程序。这些程序都被扫描脚本探测到了dylib劫持漏洞。

![](http://drops.javaweb.org/uploads/images/8e1f463fc5721d7d67757ec8857bf7991aeca81d.jpg)

拥有了自动挖掘程序漏洞的工具之后，下一步理所当然就是自动创建兼容的劫持dylib。回顾一下之前为了成功劫持dylib必须手动更新的两个地方。第一个就是劫持dylib的版本号必须与合法的dylib兼容。第二个（在rpath劫持的情况下）就是劫持dylib必须包含一个re-export（LC_REEXPORT_DYLIB）加载命令指向合法的dylib，并保证所有的需要的符号都被导出。

可以非常直接的自动将生成的dylib修改以满足上述两个条件。下面介绍第二个Python脚本，createHijacker.py（下载地址[15](http://drops.javaweb.org/uploads/images/56731aba984a005efaa14401bbb8de1c1eddaac2.jpg)）,用来完成这个功能。首先，脚本寻找并解析目标dylib（漏洞程序加载的那个合法dylib）的相关LC_ID_DYLIB加载命令。这样可以提取到需要的版本兼容信息。然后对劫持dylib进行同样的解析，直到找到LC_ID_DYLIB命令。脚本接下来就用提取到的兼容信息来更新劫持脚本的相同加载命令，以保证版本信息的兼容。接下来，在更新劫持dylib的LC_REEXPORT_DYLIB加载命令，让其指向目标dylib。虽然可以手工完成更新LC_REEXPORT_DYLIB加载命令，但是用install_name_tool命令来处理会更加简单。

图34显示了用Python脚本自动处理生成的劫持dylib。该劫持dylib用来劫持rpathApp.app示例程序。

![](http://drops.javaweb.org/uploads/images/35c3dff53d59628ae544753c4205f5c4506beff5.jpg)

图34 劫持dylib的自动生成

Dylib劫持可以用来实现大量的恶意行为。本文知识涉及到其中一些，包括隐秘驻留，加载时注入，安全防护绕过，甚至是绕过Gatekeeper。这些攻击虽然极具危险性，但是仅仅通过一个简单的植入恶意dylib就能实施，只是需要利用一下系统加载器的合法行为。正是因此，这看起来有点微不足道，所有不大可能被修补，或者是被个人安全软件所检测到。

利用dylib劫持来实现长期隐秘驻留是该攻击最重要的功能之一。如果一个漏洞程序可以在系统开机或者用户登录的时候自动运行，那么一个本地攻击者就可以利用dylib劫持来实现恶意代码的开机自动执行。除了能够实现一种新的驻留方式，这种方式还能很好的隐藏攻击者。首先，只需要植入一个简单文件，而不需要更新系统组件（比如启动配置文件或签名的系统文件）。这是很重要的，因为这些组件基本都是被安全软件监控，或者需要简单验证的。其次，攻击者的dylib将会注入到一个可信进程里面，这使得它很难被检测到，因为不会表现出很大不同。

当然，实现这样隐秘和优雅的驻留，需要一个随系统启动的含有劫持漏洞的程序。苹果公司的iCloud Photo Stream Agent（/Applications/iPhoto.app/Contents/Library/LoginItems/ PhotoStreamAgent.app）程序会在一个用户登录时候自动启动，以此来将本地数据与云端同步。幸运的是，这个程序包含多run-path搜索目录并且在第一个run-path搜索目录找到不到目标文件的@rpath导入。换句话说，这就是极好的劫持攻击目标。

![](http://drops.javaweb.org/uploads/images/4c2bb25d31fa7f896a1c7579ea14cb7eb1bd9a9e.jpg)

图35  有漏洞的苹果公司相片流工具

使用createHijacker.py脚本，可以轻松的将恶意劫持dylib配置更新为兼容目标dylib和程序。在这个例子中需要注意，因为存在漏洞的导入（'PhotoFoundation'）是在一个框架打包文件（framework bundle 见图35）里，所以需要重新生成打包文件然后再放置到第一个run-path搜索目录（/Applications/iPhoto.app/Contents/Library/LoginItems/）。当打包文件放置正确，且恶意劫持dylib（重命名为'PhotoFoundation'）也放置在第一个run-path搜索目录中时，加载器就会在iCloud Photo Stream Agent启动的时候发现和加载恶意dylib了。因为程序是由系统启动的，所以劫持dylib在重启后可以隐秘的启动。

![](http://drops.javaweb.org/uploads/images/975f64ea78dcdcec12a3e242448dc03af1b80846.jpg)

图36 劫持苹果的照片流工具达到驻留的目的

关于驻留的最后一个需要注意的地方是，如果没有发现随系统自动启动的漏洞程序时，任何可以被用户手动启动的含有漏洞的程序（如浏览器，邮件客户端等）都可以成为目标。换句话说，任何正常的程序都可以简单的通过各种方法变为自启动程序（比如注册成为Login Item等）。然后在再进行利用。虽然这种方式增加了攻击暴露的可能性，但是攻击者的dylib可以避免任何界面出现在屏幕上。因为，绝大多数用户都不会注意到一个合法的程序在后台自动运行。

进程注入，或者强制一个进程加载一个动态库是dylib劫持攻击的另外一种很有用的攻击方法。在本文中，‘injection’代表加载时注入（程序任意时间启动的时候）而并不是运行时注入。后者看起来应该是更有威力，但是前者更简单，大多数时候也能造成同样规模的破坏。

利用dylib劫持来强制一个外部进程加载一个恶意dylib是一种既隐秘又威力十足的技术。和其他大的dylib劫持攻击技术比起来，它不需要任何对操作系统组件或程序进行修改（例如对目标进程文件打补丁）。同时，因为植入的dylib都会在目标进程每次启动的时候自动隐秘的加载到进程中，攻击者也不需要任何监视器组件（来发现目标进程是否启动，然后注入恶意库）。因为攻击者仅仅只需要植入恶意劫持dylib，比复杂的运行时注入要简单的多。最后，因为注入技术是使用的操作系统加载器的合法功能，所以不容易被安全软件探测到（安全软件通过监视‘inter-process’API来阻止远程进程注入）。

Xcode是苹果公司的集成开发环境（IDE）。开发者使用它来开发OS X和iOS程序。因此，它是高级黑客喜欢的目标，高级黑客也许希望注入代码到IDE中然后悄悄的感染开发者的产品（例如，作为一个创新的自动恶意软件传播模式）。Xcode和几种它的辅助工具都存在dylib劫持攻击的漏洞。特别是，run-path-dependent dylibs，比如DVTFoundation就没有在Xcode的第一个run-path搜索目录里面，见图37。

![](http://drops.javaweb.org/uploads/images/0ed72b542082e042e4ce7346a496c3f633fbc926.jpg)

图37 苹果IDE Xcode的漏洞

对Xcode的进程注入可以直截了当的完成。首先，配置好一个劫持dylib，它的版本信息是兼容的，它的re-exported指向合法DVTFoundation的所有符号。然后，将配置好的dylib拷贝到/Applications/Xcode.app/Contents/Frameworks/DVTFoundation.framework/Versions/A/（Frameworks/是第一个run-path搜索目录）。现在，一旦Xcode启动，恶意代码就会自动加载。接下来就可以自由的执行任何功能了，比如中断编译请求，悄悄的植入恶意代码到最终的产品中。

像Ken Thompson在他的文章‘Reflections on Trusting Trust’【16】中写到的一样，当你不能信任你的编译器的时候，你都不能信任你自己编写的代码。

![](http://drops.javaweb.org/uploads/images/ae5781d5fad54b33cc6f56e8c79b508d4a1290d1.jpg)

图38 通过dylib劫持来进程注入。

除了隐秘驻留和加载时注入，dylib劫持还可以用于个人安全软件的绕过。具体是通过dylib劫持，攻击者可以使一个受信任的进程自动加载恶意代码，然后在执行一些之前被阻止或警告的行为，现在就不会被检测到了。

个人安全产品（PSPs）通过特征码，启发式行为分析来探测恶意代码，或者简单的在用户执行某个动作时弹出以一个警告。因为dylib劫持是一个比较新的利用系统合法功能的技术，基于特征码和基于启发的产品都能轻易的被绕过。像防火墙一类的安全产品，警告用户任何从一个未知进程发起的出口连接，会对攻击者造成一些挑战。Dylib劫持也可以轻易的击败这类产品。

个人防火墙在OS X平台上很流行。它们通常使用一些算法，完全信任从已知程序发出的出口网络连接，对未知和不信任进程的网络活动向用户给出警告。这确实对以探测基础的恶意软件很有效，但是高级黑客可以轻松的找到它们的弱点：信任。之前提到了，一般来说这类产品都拥有一个默认的配置规则，允许用户创建新的规则给已知的受信的进程（例如：允许任何来自进程X的出口连接）。但是这要求合法的功能不能被破坏，如果一个攻击者可以植入恶意代码到一个可信进程里，这些代码就继承了进程的信任属性，因此防火墙将允许它的出口连接。

GPG工具【17】是一个OS X平台下的信息加密工具，提供密钥管理，发送加密邮件，或者通过插件，提供加密服务到任意程序。不幸的是，它也存在dylib劫持漏洞。

![](http://drops.javaweb.org/uploads/images/d94ffe1f680bc426a11209dead7327e2fca44e11.jpg)

图39 GPG工具集中存在漏洞的keychain程序

因为GPG Keychain要求各种Internet功能（比如从密钥服务器获取密钥），所有基本上它会拥有一条‘允许任意出口连接’的规则。见图40。

![](http://drops.javaweb.org/uploads/images/977a89a43f117b18f4165f40743d250e4fba501d.jpg)

图40 GPG Keychain的访问规则

使用dylib劫持，攻击者可以将恶意的dylib加载到GPG Keychain程序的地址空间中。所以该dylib将会继承和进程一样的信任等级，因此就可以发起出口连接而不引起任何警报。测试结果说明了劫持dylib可以无限制的访问Internet。见图41。

![](http://drops.javaweb.org/uploads/images/b02ddf24c1de856783f7dbde20051dcd866b2f2f.jpg)

图41 通过dylib劫持绕过个人防火墙（LittleSnitch）

一些有防御意识的人会准确的指出，在这种情况下，可以让防火墙针对GPG Keychain规则的更加严格，以此来抵御攻击。只允许出口连接到特定的远程端口（比如已知的密钥服务器）。然而，还存在很多的漏洞程序，它们可以被劫持，然后同样无限制的访问网络。或者，在本例中，Little Snitch防火墙默认允许任意进程连接到iCloud.com的端口，此规则在系统层面都不能被删除。只需这一条已经足够来进行绕过防火墙了（比如，用一个远程的iCloud iDrive作一个C&C服务器）。

到此，dylib劫持方法都已经介绍了。可以看到他们是如此的有威力，优雅，并且隐秘。但是它们都需要接触到用户的电脑。然而，dylib劫持技术同样可以被远程攻击者使用，用来协助更加容易的控制一台远程电脑。

有很多办法来感染一台Mac电脑，但是最简单最可靠的是直接发送恶意内容到目标。这种比较低端的方式让用户手工去下载安装一个恶意文件。攻击者可以创造性的利用多种技术来达到目的，比如提供需要的插件，虚假升级包或补丁，虚假安全工具，或者是一个被感染的种子文件。

![](http://drops.javaweb.org/uploads/images/97654f72fe241bb368d1531f7141e5d6362275f8.jpg)

图42 伪装过的恶意内容

如果用户被欺骗下载并执行了任何恶意文件，他们就会被感染。虽然是低端技术，但是这种技术的影响却不可小觑。例如：当一款名叫Mac Defender的欺诈安全软件使用这种方式传播的时候，成千上万的OS X用户被感染，AppleCare接到超过6万次请求来处理这个感染【18】。

靠一些小骗术来感染远程目标在针对有电脑安全知识的人群来说就显得没有那么有效了。一个更可靠（尽管更加先进）的技术是当用户下载合法软件时进行中间人攻击。由于苹果商店的限制大多数软件依然是通过开发者或者公司的网站来下载。如果这样的软件通过一条不安全的连接（比如：http）来下载软件，可以接触到传输网络的攻击者可以在传输过程中感染下载的文件。当用户运行了被感染的软件，它们的电脑就会被感染，如图43所示。

![](http://drops.javaweb.org/uploads/images/e23934cdace2698439b905e5a67492a408b5a41b.jpg)

图43 中间人攻击下载过程

读者可能会想，现在是2015年了，大多数软件都是通过安全渠道下载的，对吗？不幸的是，即使是今天，大部分第三方软件都是通过不安全的渠道分发的。举个例子，在笔者的电脑上，66%的软件都是不安全渠道下载的。

![](http://drops.javaweb.org/uploads/images/c50785b3a1d38af286d57c939707ea31d837d690.jpg)

图44 笔者机器上通过http协议下载的软件

更多研究显示，几乎全部第三方OS X平台的安全软件都是使用不安全方式下载的，见图45。

![](http://drops.javaweb.org/uploads/images/80db38b86f09fee1d5581934bcb030ec6af2605d.jpg)

图45 大部分OS X平台下的安全产品的不安全的下载

苹果也意识到了这些攻击的危害，从版本OS X Lion（10.7.5）开始，Mac电脑都内置了一款安全软件GateKeeper，用来直接抵御攻击。

Gatekeeper的概念很简单，但是很有效：阻止任何不受信的软件执行。在后台实现起来还是有点复杂，但是从本文的角度来说，一个概括性的解释就足够了。当任何可执行文件下载后，会被标记上隔离属性。这个文件第一次执行时，Gatekeeper会验证这个文件。是否执行取决于用户的设置，如果这个软件没有用一个已知的苹果开发者ID签名（默认），或者不是来自苹果商店，Gatekeeper将不会允许程序执行。

![](http://drops.javaweb.org/uploads/images/56731aba984a005efaa14401bbb8de1c1eddaac2.jpg)

图46 Gatekeeper执行

随着新的版本的OS X自动安装并开启Gatekeeper之后，欺骗用户安装恶意软件的方法或是感染后的不安全下载文件（这将破坏数字签名）都基本上失效了。（当然，攻击者可能试图获得一个有效的苹果开发者证书，然后签署他们的恶意软件。然而，苹果对分发此类证书相当谨慎，而且，有一个有效的证书撤销程序，如果发现任何证书滥用就可以阻止证书。此外，如果被设置为只允许从苹果应用程序商店的软件，这种滥用的情况是不可能的。）

不幸的是，使用dylib劫持技术可以让攻击者绕过Gatekeeper来执行未签名的恶意代码，就算用户设置将设置设为仅允许来自苹果商店并签名的代码。这就重新打开之前讲述攻击方式的大门，将OS X用户又置于危险之中。

理论上，通过dylib劫持绕过Gatekeeper是合乎逻辑的。虽然Gatekeeper完全验证正在执行的软件包的内容（例如，应用程序包中的每一个文件），但是它不验证“外部”组件。

![](http://drops.javaweb.org/uploads/images/9093e91ca196ca3959800bf6c75b0e8c21a3ca9a.jpg)

图47 理论上dmg和zip文件都能绕过Gatekeeper

通常情况下，这不是一个问题，为什么一个下载（合法）的应用程序的应用程序会加载相对外部代码？（提示：相对的，不是外部文件。）

因为Gatekeeper只验证内部内容，如果一个苹果签名或来之苹果商店的程序包含一个相对外部的可以劫持的dylib，攻击者就可以绕过Gatekeeper。具体的，攻击者可以生成（或者在传输途中感染）一个.dmg或.zip文件，该文件有必要的文件夹结构以满足在外部相对的位置来包含恶意dylib。当合法程序被可信用户执行时，Gatekeeper将会验证程序包然后允许它执行。在进程加载阶段，dylib劫持会被触发，相关的外部恶意dylib会被加载，即使Gatekeeper被设置为只允许执行来自苹果商店的程序！

找到一个可以满足先决条件的漏洞程序也非常简单。Instruments.app是一个苹果签名Gatekeeper认可的程序，安装在Xcode.app的子目录中。它依赖于程序包以外的关联dylib，这些dylib就可以被劫持。

![](http://drops.javaweb.org/uploads/images/2f4cc3a8b9a1a15c1fdd6bc4969d39d9515e03ee.jpg)

图48 含有漏洞的苹果的Instruments app。

通过一个含有漏洞的可信程序，一个恶意的.dmg文件可以绕过Gatekeeper。首先，Instruments.app被打包进该文件。然后，建立一个包含恶意动态库的外部文件夹结构（CoreSimulator.framework/Versions/A/CoreSimulator）。

![](http://drops.javaweb.org/uploads/images/ae7552abad87e6b75f040e74bcd3fddb689c928f.jpg)

图49 恶意.dmg文件

为了使恶意的.dmg文件更加可信，外部文件可以设置为隐藏，在最上层的别名（和一个用户的图标）指向Instruments.app。背景也更换掉，整个文件设置为只读（所有双击的时候会自动显示）。最后的结果见图50。

![](http://drops.javaweb.org/uploads/images/913fe2e8272e7aef1dfba15e9b3d7658328a267c.jpg)

图50 最终版的恶意.dmg文件。

将恶意.dmg（虽然看起来是无害的）文件上传到一个公共的URL地址来进行测试。当进过Safari下载后执行，Gatekeeper标准的‘this is downloaded from the Internet’消息窗口弹了出来。重要的是这个消息针对任何的下载内容都会弹出，这并不是出现了什么异常。

一旦这个消息窗口被取消，恶意代码就会悄悄的同合法程序一起加载。当然，这是不应该被执行的，因为Gatekeeper的设置时最严苛的（只允许执行苹果商店下载的程序）。见图51。

![](http://drops.javaweb.org/uploads/images/d2cdcf1c02480fd3cc35560ba4fcb0a929b274ec.jpg)

图51 通过dylib劫持绕过Gatekeeper

因为恶意dylib会在程序的main方法之前加载和执行，所有可以实现不显示任何窗口。在本例中，恶意的.dmg文件伪装成一个Flash安装包，该恶意dylib可以阻止Instuments.app程序的界面弹出。取而代之的是一个合法的Flash安装包。

有了绕过Gatekeeper和加载为签名的恶意代码的技术，攻击者可以回头使用他们的老伎俩让用户安装虚假的补丁，更新包或者安装包，虚假的杀毒产品，或者被感染的盗版程序。更糟糕的是，拥有网络级攻击技术的高级黑客（可以截断不安全的连接的黑客）现在可以任意的感染合法软件下载。再也不需要担心Gatekeeper了。

0x02 防御
=====

Dylib劫持对于OS X平台来说是一种新型的有威力的攻击技术，提供给本地和远程攻击者广泛的恶意攻击方法。不幸的是尽管和苹果公司联系多次，他们也没有对此问题表示出任何兴趣。而且，也没有简单的办法来解决dylib劫持的核心问题，因为这是利用的操作系统的合法功能。无论如何，Gatekeeper的作者应该修补一下Gatekeeper来防止未签名的恶意代码执行。

用户也许想知道他们自己能做什么来保护自己。第一，直到Gatekeeper得到修复之前，不推荐下载不可信软件或者经过不安全渠道（比如：在Internet上通过HTTP）下载合法软件。重新回顾下之前的内容可以确定的一件事就是远程攻击者通过本文里介绍的方法是不能接触到一台新的电脑的。由于OS X平台上dylib劫持技术还很新，攻击者或者OS X恶意软件现在就在本地利用此技术攻击不大可能，无论如何，它也不会伤人是确定的。

为了探测本地劫持，同时发现漏洞程序，作者开发了一款新程序Dynamic Hijack Scanner(or DHS)。DHS通过扫描整个文件系统所有的运行进程来尝试发现劫持攻击和漏洞。该程序可以在objective-see.com网站下载。

![](http://drops.javaweb.org/uploads/images/abde0103335a8d6a4477581171ceee70eef360e6.jpg)

图52 Objective-see的DHS扫描器

0x03 总结
=====

DLL劫持是影响Windows系统的广为人知的攻击手段。之前人们一直认为，OS X平台对此类攻击免疫。但本文推翻了这个假设。进行了模拟dylib劫持攻击演示。通过利用弱依赖或则run-path-denpendent导入，找到了大量的苹果和第三方程序存在的漏洞，这种攻击技术可以利用到多种攻击场景，包括本地和远程攻击。从本地隐秘驻留技术到给远程攻击提供捷径的Gatekeeper绕过技术，dylib劫持技术正变为OS X平台黑客手中的有力武器。苹果公司对此种攻击技术表示无视时。现在，OS X用户只能下载安全软件和DHS一类的工具来保证自己是安全的...但仅仅是现在而已。

参考

【15】 dylibHijackScanner.py & createHijacker.py.[https://github.com/synack/](https://github.com/synack/).

【16】 Reflections on Trusting Trust.[http://cm.bell-labs.com/who/ken/trust.html](http://cm.bell-labs.com/who/ken/trust.html).

【17】 GPG Tools.[https://gpgtools.org/](https://gpgtools.org/).

【18】 Apple support to infected Mac users: ‘You cannot show the customer how to stop the process’.[https://nakedsecurity.sophos.com/2011/05/24/apple-support-to-infected-mac-users-you-cannot-show-the-customer-how-to-stop-the-process](https://nakedsecurity.sophos.com/2011/05/24/apple-support-to-infected-mac-users-you-cannot-show-the-customer-how-to-stop-the-process).
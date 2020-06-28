# AppUse(Android测试平台)用户手册 v2-2

0x00 AppUse - 概述
=====

From:https://appsec-labs.com/wp-content/uploads/2014/10/appuse_userguide_v2-2.pdf

AppUse (Android Pentest Platform Unified Standalone Environment)被设计用来作为安卓应用渗透测试平台。是安卓应用渗透测试人员的操作系统，包含一个自定义的安卓ROM，通过钩子加载，以便在运行时观察、操纵应用。

AppUse的核心是一个自定义的"hostile"安卓ROM，为应用程序的安全性测试特别打造的包含一个基于定制模拟器的运行时环境。使用rootkit-like技术，许多钩子注入到其执行引擎的核心之中，以便能够方便地操纵应用，通过"ReFrameworker"来观察。

![enter image description here](http://drops.javaweb.org/uploads/images/1fa86ed50fc0cc490636d6400125a735b068e1e4.jpg)

AppUse包含渗透测试者为了运行和测试应用需要的任何工具：安卓模拟器、开发工具、SDK、反编译器、反汇编器等 AppUse环境设计很直观，尽可能为所有普通安卓渗透测试人员和安全研究人员设计。它自带AppUse dashboard-方便用户通过UI控制整个环境。通过安装APK到设备或者拉出、反编译、调试、操控二进制文件来配置模拟器代理，任何事都可以通过点击几个按钮来完成，方便于关注所有必要的步骤，聚焦在最重要的问题上。

![enter image description here](http://drops.javaweb.org/uploads/images/123148b6f81e0b9a2d437d3806d032ba0d442fbb.jpg)

此外，AppUse预装了许多"hack me"应用，包括他们的服务端。对于渗透测试者而言，在测试几个新工具或者技术以及一些目的时，有这些应用是非常必要的。

综上所述，AppUse是如下的一个集合：

• Android Emulator • Hacking & reversing tools for Android • Development tools for Android • Custom "hostile" Android ROM loaded with hooks • ReFrameworker Android runtime manipulator • Vulnerable applications • The AppUse dashboard

0x01 AppUse-操作系统
=====

AppUse基于linux Ubuntu系统，已经装备了常见的攻击工具，可以节省时间，提高效率。

证书
--

* * *

尽管AppUse自动登录到root，但是你发现找到这些数据非常有用，在于系统交互时。

![enter image description here](http://drops.javaweb.org/uploads/images/16113f69e8567e7e536e15cc035ecd78b2766dc2.jpg)

终端和命令行工具
--------

* * *

已经为安卓安全测试人员配置了这些工具的环境变量。在研究中使用的工具，例如ADB，已经预装并且配置完毕。此外，所有的研究和攻击工具已经嵌入到PATH环境变量，所以可以在终端的任何位置访问，并且可以自动补全。

![enter image description here](http://drops.javaweb.org/uploads/images/f8019620c68b1a3e4067866fa7cf292f7597609d.jpg)

别名  新特性 在AppUse >1.8版本中已经添加了许多别名到终端中，为了帮助渗透测试人员工作得更快。

Signapk`<apk name>`

APK需要签名以便在设备上运行。由于修改APK将使得原来的APK签名不再有效，研究人员需要签名APK。可以通过位于 /AppUse/Pentest/SignAPK/sign.sh的SignAPK做到，该脚本需要参数：[existing_apk]，因此该脚本将会在当前目录创建一个名为_signed.apk的签名APK，并删除旧的未签名APK

别名：signapk `<apk path>`是执行/AppUse/Pentest/SignApk/sign.sh的更短的方式，以便签名被修改的apk文件。

![enter image description here](http://drops.javaweb.org/uploads/images/961d122b787bf2547dae79d9f2fd1a8c699927b9.jpg)

图4:SignApk

Search`<string>`

search别名是一个python脚本，使用bash在文件中搜索字符串，在渗透测试人员在文件中搜索一个方法名获取其他字符串时非常有用。

![enter image description here](http://drops.javaweb.org/uploads/images/714a54ac3bf2a9340a79790270212f566a1007a7.jpg)

Rootdevice Rootdevice是执行/AppUse/Emulator/rooting/root.sh root设备的快速方式。

Jdgui`<jar file>`  Jdgui是执行JD-GUI的快速方式。

开发工具
----

* * *

AppUse环境包含了安卓开发以及调试工具，可以方便执行应用程序的渗透测试。这个环境预装 了ECLIPSE和ADT，用于编写应用程序或者exp。此外，这个环境预装了IPYTHON，方便开发python脚本，帮助在一些特殊场景中进行测试。

![enter image description here](http://drops.javaweb.org/uploads/images/cceceb8eb1b8faacec09adb63515fc3b5bdff77f.jpg)

AppUse目录结构
----------

* * *

AppUse位于/AppUse目录，这个目录包含了许多目录：

```
.Android --包含两个目录，一个目录包含SDK，另一个是ReFrameworker，包含整个ReFrameworker 平台
.Emulator --包含模拟器目录，例如sdcard，wallpaper，rooting等
Logs -- 包含AppUse dashboard日志文件
Pentest -- 包含测试工具的多个目录，例如Burp, apktool, jdgui,mercury, dex2jar等
Targets -- 最重要的目录，包含目标在接下来的段落中描述的APK/应用程序数据目录

```

Targets 目录
----------

* * *

AppUse的主要目标是组织渗透测试者的所有工作，为了实现这个目标，在AppUse中应用的所有工作都将保存在同一个目录--Targets目录。Targets目录包括所有目标应用目录，包含了在dashboard中所有工具的输出。

在反转部分或者应用数据部分使用的特征之一，目标apk/app的数据将会输出在Targets目录中目标app的目录下。例如，如果你反编译AppSec.DropBoxPoc-1.apk，那么输出将会保存在/AppUse/Targets/ AppSec.DropBoxPoc-1/disassembled/ 

![enter image description here](http://drops.javaweb.org/uploads/images/44e8628aab89679dd5e74d9186c2d0e189b1969d.jpg)

Pentest目录
---------

* * *

AppUse是一个不断发展的项目。一个主要的目的就是总是赶上最新的工具，使研究者能够对一个给定的应用实现完全覆盖的攻击。 AppUse目录结构包含了/Pentest目录，包含了所有工具。该文件夹的简要视图如下：

![enter image description here](http://drops.javaweb.org/uploads/images/5eda1692faa4c1c5b037ca8598e0b2cf88755f2f.jpg)

你可能注意到，并不是所有的工具都呈现在DashBorad中。Dashboard有如此多的功能以至于在标准的研究中可以覆盖所有研究者的需求，有些则仍在开发之中，尚未嵌入到Dashboard中。AppUse不会阻止你使用这些，因为我们知道，对于有些研究者，他们需要这些。这些都可以在/Pentest目录下可以找到。

例如，如下展示了运行中的APK Analyzer：

![enter image description here](http://drops.javaweb.org/uploads/images/01a3075878ee0812f2e64ff02de1c9c838245cbb.jpg)

0x02 AppUse - Dashboard
=====

概述

Dashboard 是AppUse测试环境的心脏。Dashboard是一个组织了测试工具和运行时环境的GUI视图，Dashboard会把各种不同工具链接的所有数据放在一起，并在其特殊的功能中节省宝贵时间，将几个动作链接在一起，并且在文档中进一步论证。

双击桌面上的链接来运行Dashboard，其将会立即运行：

![enter image description here](http://drops.javaweb.org/uploads/images/6cf65d1eac1bcbbd8dfa18067115194f07871b97.jpg)

Structure：

Dashboard分布了7个选项，每一个都有特殊目的。

General：

Dashboard中，以APK为主。AppUse Dashboard意味着允许研究者通过单击动作开始工作。为了实现这个目的，Dashboards被设计用来在APK上操作，来调用它的其他动作。

Load APK

Load APK是最基本的操作，Load APK将从文件系统加载一个APK到Dashboard中，以使得Dashboard执行其动作。当开始一个应用程序的研究，这是第一个必要的动作。加载的APK将会被复制到/AppUse/Targets/`<apk_name>`/目录。

![enter image description here](http://drops.javaweb.org/uploads/images/a2955168f0c1f67bf5d5239445ea9b472148cb38.jpg)

Install APK

在用户从文件系统加载了APK后，Install APK按钮将会出现。这个按钮允许渗透测试者安装一个APK到一个正在运行的模拟器中。为防止应用已经安装过，Dashboard会询问是否重装或者卸载并再次安装该应用。

![enter image description here](http://drops.javaweb.org/uploads/images/2aef8a861463a66e8602817a5e0ef36e4873f828.jpg)

目前设备状态

设备状态图片展示了目前连接的设备状态。在设备连接上AppUse VM时会变绿，标签为ON，在没有设备连接时会变红，便签为OFF。它帮助你在设备连接上了时，避免使用Check Device按钮来获取快速指示。

![enter image description here](http://drops.javaweb.org/uploads/images/80068a94209983b88a910e0a22b6f70c182b1d97.jpg)

Help

帮助图片允许你得到当前选择部分的信息。通过点击它，AppUse 当前部分的用户指南将会在浏览器中打开，这将帮助你明白Dashboard中每个按钮的目的。

![enter image description here](http://drops.javaweb.org/uploads/images/b782f4c42895afdb853d818fc98df9509d9e9311.jpg)

Android Device 实现Android部分是为了执行模拟器上的动作，通过单击可以实现如下动作：

Launch Emulator  Launch Emulator 按钮是为了打开AppUse模拟器

Restart ADB Restart ADB是为了重启ADB服务，以便让AppUse能够识别设备，万一ADB服务器起不来。在AppUse1.8中实现了一个自动机制来检查服务是否为down并且重启ADB。

Install Burp Certificate 新功能

Install Burp Certificate按钮允许AppUse在模拟器中安装Burp证书，在下一个版本中，它将在任何设备连接AppUse VM时安装burp证书。

Root Device 模拟器上的Root权限在渗透测试中会派上用场，但是通常会消耗很多时间。AppUse Dashboard有一个内建的选项来自动root模拟器，通过点击按钮。

![enter image description here](http://drops.javaweb.org/uploads/images/79cd48ab0472616c93d65920bd754fe96ab4ae41.jpg)

Take Screenshot 

Take Screenshot 用来实现对模拟器截图，截图保存在/AppUse/Emulator/Screenshots/ 目录

![enter image description here](http://drops.javaweb.org/uploads/images/8be97119f3a0fc68573a7aca9413492e9072e46f.jpg)

Open ADB shell 

Open ADB shell 按钮用来使渗透测试者更简单打开ADB shll 终端

![enter image description here](http://drops.javaweb.org/uploads/images/aa89065e9bbf99fd2e4e512e151354973fd10cd3.jpg)

Proxy Settings  新特性

Proxy Settings面板用来是渗透测试者更容易root设备，以及在设备上打开代理。AppUse预装了应用与Dashboard通信，允许渗透测试者在模拟器中通过单击ON/OFF打开关闭代理。设置代理需要IP和PORT，另外一个重要特性是允许决定重定向端口。万一目标应用使用二进制协议，它可能需要设置特定端口或者监听所有端口，并且中断二进制协议流量。如果设备没有root，AppUse将会自动root。

![enter image description here](http://drops.javaweb.org/uploads/images/a8f7c98115097423c28aabe5de9966d01a95a1cf.jpg)

TOOLS 工具部分是用来给渗透测试者在测试中快速访问这些有用的工具。通过单击可能执行如下动作：

Launch Burp 运行Burp代理

Launch Firefox 运行火狐浏览器

Launch Wireshark  AppUse预装了Wireshark ，在Dashboard中运行。

Launch Eclipse 运行Eclipse IDE

Launch NetBeans 运行NetBeans 8.0，可以调试应用

Launch IDA 运行IDA，可以反汇编二进制文件

Launch Mercury client 运行Mercury Console客户端。在设备中通过Mercury Agent打开服务后，接下来执行端口转发，然后打开Mercury Console。所有操作可以通过单击Launch Mercury Client完成

![enter image description here](http://drops.javaweb.org/uploads/images/53fca9e1854f9ab8fa51cf183e8690b6b2ad4737.jpg)

Launch SQLLite Browser Launch SQLLite Browser运行 SQLLite Browser，查看数据库文件

Launch JD-GUI  Launch JD-GUI 运行JD-GUI ，查看JAR文件源代码

Open Terminal  Open Terminal 打开shell，在系统中执行操作

Download APK from Google Play  从Google Play下载应用，因为在模拟器中没有google

Open AppUse Directory 打开AppUse目录，查看文件和操作

Reversing AppUse包括解码和逆向APK的最先进的工具。一旦APK被加载到Dashboard中，所有工具已经配置好，可以直接使用，可以实现完全覆盖。 逆向部分用来执行像从设备拉出APK，反编译，拆解，组装，转换到APK模式等。这部分将使得工作更快，通过单击执行如下动作：

Reverse APK Content 用来实现三个动作：反编译，拆解，unzip

Decompile (JD-GUI)  JD-GUI是一个拆解jar文件的框架。一旦.dex文件必须被转换为.jar文件，JD-GUI将准备好来拆解这些代码。通过使用JD-GUI，能够审计应用源码，发现隐藏秘密和逻辑。 Decompile按钮用来帮助渗透测试者反编译目标APK，通过dex2Jar转换成JAR文件，然后使用JD-GUI查看源代码。所有这些都通过单击按钮执行。AppUse将拉住APK，反编译它，通过JD-GUI打开目标apk的jar文件。

![enter image description here](http://drops.javaweb.org/uploads/images/ca7721d9b797c138a06b1caea4a43fd8b34e945e.jpg)

Decompile (Luyten) 和上面的按钮一样实现相同操作，但是这个通过Luyten decompiler打开jar文件

 Save Java Sources 用来保存目标APK的java源代码到目标apk目录的decompiled目录

Dissemble (Baksmali)  Baksmali是一个反编译apk Dalvik字节码的工具，研究者能够来查看应用程序的Dalvik编码，并且通过人类可读的格式来修改它。 这个按钮用来帮助渗透测试人员通过多个命令反编译APK，所有操作通过单击实现： 一旦在APK上使用了baksmali，/AppUse/Targets/`<Targeted APK>`/Dissasembled/目录会包含所有的Dalvik编译代码，然后渗透测试者能够修改并提供额外的指令

![enter image description here](http://drops.javaweb.org/uploads/images/6bce40a2daae23fa3323b0438d354573b8416ac2.jpg)

Reassemble(smali)  Baksmali让研究者能够得到人类可读的dalvik编译代码，并且有机会以任意编辑器去编辑它。Smali是完成打包代码的工具。 通过Smali，研究者能够改变应用的编译代码，并且重新打包为新的.dex文件。一旦行的.dex文件应用到APK文件上，这些更新会作为补丁打上，一旦APK被安装，这些更新会实时应用。使用这个特性可以提升安全研究到一个新的水平。

这个按钮是通过Baksmali Dissasemble编译反编译的目录。在修改了Smali代码后，希望转换为APK文件， 并且安装它。所有这些可以通过单击这个按钮实现。AppUse将会编译这个反编译目录，并且创建一个签名APK。

View Manifest 新特性 这个按钮将打开APK的manifest文件。它将提取apk的menifest并在firefox中打开。

Debug Mode 新特性 这个按钮通过自动进入调试模式，来调试APK。通过单击，这个应用将会反编译APK，修改menifest，再编译，并且签名。

Open Target Directory 这个按钮用来打开目标应用的目录，用来查看文件并执行动作。

Application Data  应用数据部分是用来访问目标应用的文件。Load APK用来加载应用的数据目录到列表中，用户可以过滤目录名。渗透测试者将从列表中选择目标应用的目录，然后单击Load Folder，在目标目录上执行动作。通过单击实现如下动作：

View File 用来查看目标应用内的文件。加载目录到一个树形时候后，选择一个文件，点击cat文件，查看其内容。

Edit File 用来修改目标应用内的文件。通过单击AppUse，将会拉住文件，并在编辑器中打开。在修改并保存后，将会自动推送到设备中。

Pull File/Folder 用来从目标应用中拉住文件或目录。树形视图允许更快地查看和选择文件和目录。

Extract Databases 用来查看数据库文件。与拉住DB文件，并在SQLLITE 浏览器中打开每一个不同，单击AppUse将会拉住DB文件，通过Doxygen解析数据库数据为HTML，允许其查看每一个文件中的每一张表。

Open App Data Directory 用来打开目标应用目录，查看文件，执行动作。

ReFrameworker 

Launch ReFrameworker  运行ReFrameworker平台，将在文档后面阐述。

Enable/Disable ReFrameworker  用来在模拟器中替换JAR文件，并重启。允许ReFramewoker在模拟器中加载钩子。

Add Internet Permissions 为了给Reframeworker的 dashboard发送消息，它创建了一个web请求从应用到dashboard。如果在Reframeworker的 dashboard发送send或者proxy，并且应用没有web请求的权限，它将不会工作。这个按钮给予应用临时Internet权限。

Vulnerable Apps  训练部分用来打开服务端的训练应用，可能会打开HackMePal HTTP/S servers、GoatDroid、 ExploitMe。

AppUse - Runtime Modifications and Inspection via AppSec ReFrameworker  AppUse中的模拟器已修改来满足渗透测试者的需求。它配置了一个预制的ROM，预装了工具与系统交互。

AppUse的核心是一个自定义的hostile安卓ROM，专门为应用安全测试而建立，包含了一个修改的实时环境，运行定制模拟器。使用rootkit-like技术，许多钩子会注入到它执行引擎的核心中，因此，应用程序可以很容易被操控，并通过叫做ReFrameworker的命令&控制器来观察它。

How it works – an overview  安卓运行时在代码关键部分配置了许多钩子，这些构造寻找一个叫做Reframeworker.xml的文件，该文件位于/data/system。在每个应用运行时，每当一个被勾住的运行时方法被调用，都会与包含的规则("items")一起加载ReFrameworker的配置文件，并作出相应的反应。 与它的规则一起管理配置文件已经通过ReFrameworker dashboard做到，使用面板，你可以定义一系列Android运行时会遵从的规则。米那般将会生成一个配置文件，运行时环境将随后解析并执行。 例如，它随着加载一个可以是本地、也可以是直连安卓设备上的配置文件一起启动。在点击load config按钮后，面板将会标记所有加载的规则，并允许用户enable/disable或者配置它们、

![enter image description here](http://drops.javaweb.org/uploads/images/3ea5541f4a2137d3881dfef69c432cd981668e10.jpg)

在文件被加载后，面板将会用粗体标记所有定义的规则，并且标记哪些使用了的规则为绿色。

然后用户可以选择运行时的行为。例如，能够对重要消息打开嗅探、绕过某些逻辑，执行字符串替换，给ReFrameworker dashboard发送数据等。

接下来用户可以保存行的配置。如果用户选择保存到设备中，设备立即能够遵循这些规则来执行。

![enter image description here](http://drops.javaweb.org/uploads/images/99bbdb433f4a24d7376cd7e9d567e98495d93457.jpg)

通过点击规则的目标可以配置每个规则的行为，可以从子菜单中选择配置，

![enter image description here](http://drops.javaweb.org/uploads/images/30cf415ec2a0a5be57938e97bdea2bcf0278edd0.jpg)

然后出现一个新的窗口，包含规则的值。每一个规则都有如下性能：

```
• Name – the name of the rule 
• Enabled – is it enabled? 
• Calling method – the name of the runtime method upon which this rule should apply 
• Mode – can have 3 possible values – Send, Proxy, or Modify  
o     Send – send the hooked content to the ReFrameworker dashboard 
o     Proxy -  let the user control the value of the hooked content by using a proxy-like UI 
o     Modify – replace a particular content with another content 

• Value – specify the condition for the hooked content. An asterisk (*) means “always.” 
• toValue -  specify the action for the hooked content. An asterisk (*) means “always.” 

```

如下是配置规则的例子：

![enter image description here](http://drops.javaweb.org/uploads/images/05db7d68fdc72eb3adf6e1b217c802be244b14b2.jpg)

如下像是生成了新的规则后的配置文件。如下是Reframeworker.xml配置的例子：

![enter image description here](http://drops.javaweb.org/uploads/images/9dc2d08a9470bed6be34e34d0d410f0eb3a19ff5.jpg)

这里的思路是把这个文件烦躁运行时的特殊地点，我们的钩子能够找到它。文件路径为：/data/system/Reframeworker.xml--以便预注入的钩子将动态解析、加载、执行，使得我们可以在Android设备外部提交他们。

既然设备可以与dashboard通信，dashboard包含了一个监听器，监听从设备建立进来的通信连接。因此，dashboard有一个listener按钮：

![enter image description here](http://drops.javaweb.org/uploads/images/27fb8f07c030de8ebb9095585ef9ed40331bdbaf.jpg)

在打开监听器后，dashboard就准备接受任何进来的消息。 AppUse也有一些高级特性，让用户中断一些从内部安卓对象过来的内部消息。我们可以通过在安卓应用和运行时环境之间建立代理来实现。与实现HTTP 代理是非常类似的方式，但是只有在安卓运行时内处于一个非常低水平时才这么做。

The hooks  在一些关键地方，AppUse环境与许多钩子编译在一起。作为研究的一部分，在找到我们希望控制的有意思的地方后，例如，文件处理、通信、加密等，我们把它们放置在ReFrameworker controller中。 controller会检查是否有规则是为这个特殊位置定义的，是否与其配置运行一致。

例如，作为研究的一部分，找到有意思的地方并插入钩子，我们决定在SQLiteDatabase的executeSql插入钩子，所有请求都会经过executeSql。在这个类中插入钩子使得我们终端所有从应用发送到本地DB的sql请求。我们的钩子将会中断所有值，并执行配置文件中所有指示。

钩子通常放在重要的值中，因此如果一个规则是为一个特殊的钩子定义的，controller就能够通过它做任何事情。controller可以什么也不做，也可以发送数据到远程地方，也可以允许用户实时终端和修改值，或者可以自动替换为另外的值。

例如，这是一个预加载的钩子，勾住excuteSql方法的sql参数。实际的查询将会通过运行时环境，作为更高层应用的指示执行。

![enter image description here](http://drops.javaweb.org/uploads/images/20746689669dbfba0c90e91ec063c38f3ab34d78.jpg)

假设相关的配置规则已经被定义为proxy。现在设备每一次调用这个方法都会发送数据给代理，并将用收到的修改了的值替换原始值。

在dashboard端需要做的就是操作proxy：

![enter image description here](http://drops.javaweb.org/uploads/images/cec68563c6855524b142afefc122e7ad8812d7ca.jpg)

当收到消息后，proxy将会wake up，让用户观察并修改消息。在操作时，安卓app会一直等待相应。

![enter image description here](http://drops.javaweb.org/uploads/images/1d1a95333aad7da552eca4e2e4846e93627d5388.jpg)

Configuration examples 

Example #1--发送所有执行了的命令的值到Dashboard

![enter image description here](http://drops.javaweb.org/uploads/images/c095aaf9dbba95cadbdfdc5ac7ef09a19a3764fc.jpg)

解释--在希望发送数据时设置为send，值是_，意味着我们希望发送所有命令。toValue在这个情景中没有使用，但是也设置为_。calling method设置为被勾住的方法。

Example #2  -- 发送所有执行SQL请求的值到Dashboard

![enter image description here](http://drops.wooyun.org/wp-content/uploads/2015/04/image065.jpg)

解释：calling method设置为与sql 查询相关的特殊方法，其他值保持一致。

Example #3 --proxy (break and modify) 所有执行的SQL请求值到dashboard

![enter image description here](http://drops.javaweb.org/uploads/images/5591256993aee07ee72e1c9b4d1ff19615df8c95.jpg)

解释：既然我们希望实时修改数据，将method设置为proxy，其他值保持一致。

Example #4 -- 信任所有证书

![enter image description here](http://drops.javaweb.org/uploads/images/9d48e229acc8f4878c47c6f2f1fface1bb716420.jpg)

解释：既然我们希望替换所有hooked的值(尤其是无论证书是否信任的布尔值)。既然我们希望替换所有的值，那么value设置为*。既然希望总是相信证书，那么toValue设置为true。method设置近似hooked方法。

Example #5 --禁用hostname 识别

![enter image description here](http://drops.javaweb.org/uploads/images/57b10278b94dc70d6642b2b1eb2659279ddec89a.jpg)

解释：与前面例子相当相似。唯一不同的是calling method的值为hostname verification

Example #6 -- 替换手机IMEI为另外一个值

![enter image description here](http://drops.javaweb.org/uploads/images/14b0318b5915a17749350580b46caf9b5033b4ed.jpg)

解释：既然我们希望替换数据，那么mode设置为modify，value设置为*，用来替换所有值，toValue设置为我们在这个例子中希望设置的值11111111，callingmethod与hooked方法类似。

Example #7 --   the proxy (break and modify) is the value phone IMEI number

![enter image description here](http://drops.javaweb.org/uploads/images/898356d9b153673b7b4aa912ae72cb697fce070b.jpg)

解释：既然我们希望实时修改数据，那么mode设置为proxy，其他值保持一致。

当然，这只是很剪短地介绍了ReFrameworker，还有AppUse可以管理的规则，每一个都有许多不同的设置。
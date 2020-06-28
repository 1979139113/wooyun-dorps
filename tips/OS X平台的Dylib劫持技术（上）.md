# OS X平台的Dylib劫持技术（上）

原文：[https://www.virusbtn.com/virusbulletin/archive/2015/03/vb201503-dylib-hijacking](https://www.virusbtn.com/virusbulletin/archive/2015/03/vb201503-dylib-hijacking)

作者：Patrick Wardleku

翻译水平有限，请理解请指正

DLL劫持是一项广为人知的攻击技术，一直以来被认为只会影响到Windows系统。然而本文将会介绍OS X系统同样存在着动态链接库劫持。通过利用OS X动态库loader的未文档化的技术和几种特性，攻击者将精心制作的包含恶意代码的动态链接库加载进存在漏洞的程序中。通过这种方法，攻击者可以实现多种恶意行为，包括：隐秘驻留，进程加载时注入，绕过安全防护软件，当然还包括Gatekeeper的绕过(提供远程注入的机会)。因为攻击者利用的是操作系统提供的合法函数调用，所以只能尽力防御，不可能被完全修补。本文将要提供用于发现存在漏洞二进制文件的技术和工具，同时也可以用来探测是否有劫持已经发生。

0x01 背景
=====

在介绍OS X平台下的dynamic library（dylib）劫持攻击细节之前，我们先介绍下Windows平台下的dynamic link library（DLL）劫持。因为这两种攻击在概念上很相似。通过学习成熟的Windows平台攻击方法可以帮助理解OS X平台下的方法。

微软给出的DLL劫持定义是： “当一个程序动态加载一个DLL时，如果没有给出完整的路径，Windows系统会试着在默认的一些目录下搜索该DLL。如果一个攻击者获得了其中一个目录的控制权，他就可以让程序加载一个恶意的DLL文件来代替原来的DLL。”【1】

需要指出的是，Windows加载器的默认搜索路径先搜索一些特定目录（如程序所在目录或当前工作目录），再搜索Windows系统目录。这种处理策略导致了漏洞的产生。比如，一个程序试图加载一个没有完整路径的系统库，只给出了名字。在这时，一个攻击者在之前的搜索目录中植入了一个恶意同名DLL。这样Windows加载器会先发现恶意的DLL而不是原来的合法DLL，并且将其加载到漏洞程序中。

下面的图1和图2进行了说明，一个存在漏洞的程序被一个恶意DLL劫持了。该恶意DLL放在当前工作目录中。

![](http://drops.javaweb.org/uploads/images/e658fa64bb1fc6963cc7db9bac92beb89a8d2834.jpg)

图1  加载正常系统DLL

![](http://drops.javaweb.org/uploads/images/8e9d767c8e7216d8bf2a6b4385a5a6260522fca2.jpg)

图2 加载攻击者的恶意DLL

DLL劫持攻击最初形成广泛影响是在2010年，然后迅速的得到了媒体和黑客的关注。起了一些如“二进制植入”，“不安全库加载”，“DLL预制攻击”等名字。发现该漏洞的人被广泛认为是H.D.Moore【2】,【3】。然而NSA是最先发现这个漏洞的，早在Moore发现的12年前，1998年，在NSA的非机密文件‘Windows NT Security Guidelines'中，NSA就描述DLL劫持漏洞并提出了警告：

'攻击者无法插入一个虚假的同名DLL在搜索器搜索到合法的DLL之前，这是非常重要的’【4】

对于一个攻击者来说，DLL劫持可以使用于多种场合。比如，攻击者可以让恶意库安静的启动加载（不改变任何注册信息和其他系统组件），权限可以得到提升，甚至进行远程感染。

恶意软件作者迅速的认识到了DLL劫持的优势。在一篇博客文章”What the fxsst?“【5】里面，Mandiant公司的研究者描述了他们怎么发现了大量不同不相关的恶意软件样本都叫‘fxsst.dll’。通过进一步的研究，他们发现这些样本都是利用了一个存在于Windows shell（Explorer.exe）里面的DLL劫持漏洞。该漏洞提供了一个隐秘驻留系统的方法。具体原因是，因为Explorer.exe是安装在C:\Windows目录下，在这个目录下植入一个名为fxsst.dll的恶意动态库文件，将会成功让恶意dll驻留在系统，因为加载器会在搜索真正fxsst.dll存在的系统目录之前搜索这个程序目录。

另一个恶意软件利用DLL劫持技术的例子可以在泄露的银行木马‘Carberp’]【6】中找到。源代码显示该恶意软件是通过sysprep.exe（图3）的DLL劫持来绕过UAC的。这个程序是一个自动提权的进程，即不需要任何UAC请求就可以获得更高的权限。不幸的是，它存在着DLL劫持漏洞，被攻击者利用来加载恶意的DLL（cryptbase.dll）【7】。

![](http://drops.javaweb.org/uploads/images/8c9c7882acc6afaa18fa5ef6368828751c8fb9b2.jpg)

图3 Carberp利用DLL劫持来绕过UAC

最近，DLL劫持在Windows平台上已经很少见了。微软迅速采取应对攻击，修补易受攻击的应用程序，并详细说明如何避免这个问题（例如，对需要加载的DLL指定一个绝对路径）【8】。然而，操作系统级的防御措施还是需要的，比如通过safedllsearchmode或cwdillegalindllsearch注册表项启用，防止大多数DLL劫持。

0x02 Dylib劫持
=====

曾经以为动态库劫持只一个Windows平台独有的问题。但是，在2010年，一个机敏的StackOverflow用户指出来，‘任何允许动态链接外部库的操作系统在理论上都是有漏洞的’【9】。直到2015年他才证明了这个观点的正确性。本文将揭示在OS X平台下具有毁灭性的动态库劫持攻击技术。

本文研究的目的是揭示OS X平台是否存在动态库攻击的漏洞。进一步说就是，本研究需要回答如下的问题：在OS X平台下，是否攻击者可以植入一个可以自动被加载器加载到漏洞程序的恶意动态库。假设OS X平台下的劫持攻击可以和Windows平台下的DLL劫持一样，能够让攻击者实现大量的攻击目的。比如：隐秘驻留，加载时注入，安全软件绕过，也许还有可以远程感染。

需要指出的是，本研究是基于一些限制条件的。第一，成功被限制在不允许对系统做出任何修改情况下，除了创建文件（或者是文件夹）。换一个方式说，本研究忽略了那些要求对特殊二进制文件进行修改（例如补丁）或修改系统配置文件（比如‘auto-run’plists等）的攻击环境要求。这样的攻击都被广为人知，容易去防止和探测，所以被忽略掉。本研究试图寻找到一个独立于用户环境的劫持方法。OS X也提供了各种合法的方式加载动态库，可以让加载器强制自动加载恶意库到目标进程。这些方法，比如设置DYLD_INSERT_LIBRARIES环境变量，是用户特定的，也是众所周知的，容易被检测到。所以我们也不进行研究，直接忽略了。

本研究从研究分析OS X的动态连接器和加载器开始：dyld。这个二进制文件在/usr/bin目录下，提供标准的加载器和连接器的功能，寻找，加载，连接动态库。

因为苹果公司曾让dyld开源过【10】，所以分析起来就很简单直接。比如，通过阅读源代码可以很好的理解dyld作为一个可执行程序被加载的过程，包括它所依赖的库加载和连接的过程。下面几点简要的总结了dyld开始的步骤（主要关注与本文研究的相关的）

1.  当任何一个新的进程开始时，内核设置用户模式的入口点到__dyld_start（dyldStartup.s）。该函数简单的设置stack然后跳转到dyldbootstrap::start()，又跳转到加载器的_main()。
2.  Dyld的_main()函数（dyld.cpp）调用link()，link()然后调用一个ImageLoader对象的link()方法来启动主程序的连接进程。
3.  ImageLoader类（ImageLoader.cpp）中可以发现很多由dyld调用来实现二进制加载逻辑的函数。比如，该类中包含link()方法。当被调用时这个方法又调用对象的recursiveLoadLibraries()方法来进行了所有需求动态库的加载。
4.  ImageLoader的recursiveLoadLibraries()方法确定所有需要的库，然后调用context.loadLibrary()来逐个加载。context对象是一个简单的结构体，包含了在方法和函数之间传递的函数指针。这个结构体的loadLibrary成员在libraryLocator()函数（dyld.cpp）中初始化，它完成的功能也只是简单的调用load()函数。
5.  load()函数（dyld.cpp）调用各种帮助函数，loadPhase0()到loadPhase5()。每一个函数都负责加载进程工作的一个具体任务。比如，解析路径或者处理会影响加载进程的环境变量。
6.  在loadPhase5()之后，loadPhase6()函数从文件系统加载需求的dylib到内存中。然后调用一个ImageLoaderMachO类的实例对象。来完成每个dylib对象Mach O文件具体的加载和连接逻辑。

理解了基础的dyld初始加载逻辑，研究重心就转移到寻找可以进行dylib劫持的逻辑上来了。特别的，研究组感兴趣的是加载器在有没有在没有找到一个dylib时，却没有报错的代码，还包括在多个位置寻找dylib的代码。如果任何一个场景在加载器中发现了，我们就有希望来进行OS X的dylib劫持攻击。

我们先研究了第一个方案，在此方案中，我们假设，一个加载器可以处理dylib没有找到的情况，一个攻击者（可以发现这种情况）可以在该位置放置一个恶意的dylib。从而，加载器就会找到放置的dylib并且不经检验的加载攻击者的恶意代码。

回顾之前，加载器调用ImageLoader类的recursiveLoadLibraries()方法来寻找和加载所有的需求的库。如图4所示，加载代码中处理dylib加载失败的代码是被try/catch块包含的。

![](http://drops.javaweb.org/uploads/images/2538342c5079d0f30849bf02e276a0146ef79741.jpg)

图4 dyilb加载失败时的处理逻辑

不出所料，处理逻辑会在加载库失败的时候抛出一个异常（含有一条信息）。有趣的是，这只有当一个名叫‘required’的变量被设置为true时，才会抛出异常。此外，源代码的注释中说明了在加载‘weak’库失败是可以的。这就说明在一些情况下，加载器加载不了一些库也是会继续正常工作的---||太棒了！

深入分析加载器程序的源代码，找到是在哪里进行‘required’变量的设置。得到结果为，ImageLoaderMacho类的doGetDependentLibraries()方法对加载命令（下面会进行描述）进行语法解析，并且通过加载命令的LC_LOAD_WEAK_DYLIB标识位来给该变量进行赋值。

加载命令是Mach-O文件格式（OS X的原生二进制文件格式）必有的组成部分。在文件中紧接着Mach-O头，它提供了不同的命令给加载器。例如，加载命令可以用来说明二进制文

![](http://drops.javaweb.org/uploads/images/df6603c86c6e0cdc2d9d88e98c3735f02a2f8e4a.jpg)

图5 设置required变量

件在内存中的布局形式，主线程的初始执行状态，和所需动态库的具体信息。可以通过工具查看编译好二进制文件的加载命令信息。比如MachOView【11】，或者/usr/bin/otool（使用-l参数）。（参见图6）

图5中的代码显示了加载器依次处理所有加载命令的过程，寻找所有声明倒入的动态库。这些加载命令的定义可以在mach-o/loader.h文件中找到。

![](http://drops.javaweb.org/uploads/images/7d87f50e5ae1adf73c1f1835ca379ae0b4f16df8.jpg)

图6 通过MachOView查看Calculator.app的加载命令

![](http://drops.javaweb.org/uploads/images/3269dcf2903192a124908e223db6edfea0161707.jpg)

图7 LC_LOAD_*加载命令的格式

对应可执行程序需要每个动态链接库，程序头都包含一个LC_LOAD_*加载命令（LC_LOAD_DYLIB，LC_LOAD_WEAK_DYLIB等）。像图4，图5中加载代码显示的一样，LC_LOAD_DYLIB加载命令声明了一个所需的动态库，通过LC_LOAD_WEAK_DYLIB声明的库就是可选的（weak）。前面一种情况（LC_LOAD_DYLIB），如果所需库没有被找到就会抛出一个异常，加载器就会放弃并结束该进程。但是如果是后面的情况（LC_LOAD_WEAK_DYLIB），动态库是可选的，如果没有发现也并没有声明影响。主程序将会继续执行。

![](http://drops.javaweb.org/uploads/images/6991caaba13809b6e31ccc872c61fb0f3109c1be.jpg)

图8 尝试加载弱（weak）库

该加载器逻辑上满足了第一个假设劫持场景的条件，因此，可以对OS X平台进行动态库劫持攻击。换句话说，如图9所示，如果一个声明的弱引用库没有找到，攻击者就可以在该位置上放置一个恶意的动态库文件。然后，加载器就会找到攻击者的动态库并且加载恶意代码到漏洞程序的进程空间。

![](http://drops.javaweb.org/uploads/images/f6437699c460e9313be2254cb9071302a3626eb5.jpg)

图9 通过恶意的‘weak’动态库进行劫持

此前说的另外一种劫持攻击是假设一个加载器在多个地方寻找动态库。在这种情况下，假设攻击者可以将一个恶意动态库放置在其中一个搜寻目录（合法的动态库在其他地方）。让加载器先找到攻击者的恶意动态库，并且不经检查直接加载攻击者的恶意动态库。

在OS X平台上，像LC_LOAD_DYLIB之类的加载命令总是将动态库的路径给出（而Windows平台只是给出动态库的名字）。正因为给出了路径，dyld加载器就不需要搜索不同的目录来寻找动态库，而是直接到指定的目录加载dylib。然而在对dyld源代码进行分析之后发现，dyld在其中一种情况下并没有进行如此的处理。

如图10所示，在dyld.cpp中的loadPhase3()函数中，有一些有趣的处理逻辑。

![](http://drops.javaweb.org/uploads/images/9dc668ce0d55fd10e75cee24fffd2532cf1a9f29.jpg)

图10 加载依赖‘rpath’的库

Dyld会循环迭代rp->paths向量来动态构建路径（存贮在‘newpath’变量中），之后调用loadPhase4（）函数。这样的做法就满足了第二种劫持场景的要求（dyld在多个位置寻找同一dylib），当然还需要进行一下路径顺序检查。

图10中，[[email protected]](http://drops.com:8000/cdn-cgi/l/email-protection)果文档，这是一个特别的加载关键字（在OS X 105，Leopard中有介绍），用来定义一个动态库为‘run-path-dependent library’【12】。苹果解释run-path-denpendent library是一种在创建时完整安装路径并不知道的依赖库。其他文档【13】和【14】等提供了更多的细节，解释了这种库所起到的作用，解释了@rpath关键字何时有效：‘frameworks and dynamic libraries to finally be built only once and be used for both system-wide installation and embedding without changes to their install names, and allowing applications to provide alternate locations for a given library, or even override the location specified for a deeply embedded library’【14】。

通过这种特性，软件开发者可以更为简单的部署复杂的程序，但同时也为动态库劫持提供了方便。一个可执行程序为了使用run-path-dependent library，需要提供给加载器运行时搜索路径列表，加载器在加载时再来寻找这些库【12】。在dyld的代码的很多地方都发现了这样的代码。包括图10里面给出的代码片段。

因为run-path-dependent library是相对新的概念，有些不为人所知，提供一个例子来说明应该是很有必要的，例子包含了run-path-dependent library和使用该库的例子程序。

一个run-path-dependentlibrary是一个安装名以‘@rpath’字符串开头的普通dylib。如图11所示，在Xcode中创建这样一个动态库只需简单的将dylib的安装目录设置为‘@rpath’。

![](http://drops.javaweb.org/uploads/images/20ef1754dd9105b552ff9e74cbd98e2acf817045.jpg)

图11 建立一个 run-path-dependent library 

当run-path-dependent library编译成功之后，检查LC_ID_DYLIB（包含了该dylib的标识信息）加载命令显示的dylib运行时路径。特别是，LC_ID_DYLIB加载命令中‘name’项，显示了该dylib的文件名（rpathLib.framework/ Versions/A/rpathLib）是以‘@rpath’开头的。如图12所示。

![](http://drops.javaweb.org/uploads/images/51edcb3f2c10317ed880190fab9cbe7f84695f23.jpg)

图12 ‘@rpath’包含在dylib的安装路径名中

构建一个加载run-path-dependent library的程序也是非常直接简单的。首先，将run-path-dependent library添加到Xcode的Libraries列表里面。然后，将run-path搜寻路径添加到‘Runpath Search Paths’列表。最后，这些搜寻目录将会在动态加载器加载库时被搜索到，以确定run-path-dependent library的具体目录。

![](http://drops.javaweb.org/uploads/images/c791a2d9e02fd59bcce224bb2b0d17f01ef180c9.jpg)

图13 @rpath库的链接设置和声明run path搜索路径

一旦应用程序被建立，dumping该程序的加载命令会显示一些与run-path依赖库相关的各种命令。一个标准的LC_LOAD_DYLIB加载命令会为需要加载的run-path-dependent dylib的关联依赖提供信息，如图14所示。

![](http://drops.javaweb.org/uploads/images/88f785e13ac60d6504ba68d31144aaf4c9796e97.jpg)

图14 @rpath库的依赖信息

在图14中，注意到安装名name项指向run-path-dependentdylib是以前缀‘@rpath’开始，并和图12中的run-path-dependent dylib的LC_ID_DYLIB命令的name值是一样的。该程序包含的与run-path-dependent dylib相关的LC_LOAD_DYLIB加载命令告诉加载器：‘我需要rapthLib dylib，但是在组建时，我不知道它的具体安装位置。请用我包含的run-path搜索路径找到并加载它。’

我们之前在Xcode中将run-path搜索路径添加进‘Runpath Search Paths’表单中。这些搜索路径会在程序中生成LC_RPATH加载命令，每条路径对应一个加载命令。查看编译好的程序可以发现包含的LC_RPATH加载命令，如图15所示。

![](http://drops.javaweb.org/uploads/images/602113cffac6f37a4d437f385bc7a2678b2883c4.jpg)

图15 加载命令中的run-path搜索路径

通过对run-path-dependent dylib和加载它的程序的理解，我们就可以更加简单的去理解dyld的源代码中负责加载动态库的那部分代码。

当一个程序启动时，dyld将会解析程序的LC_LOAD_*加载命令，加载和连接所有依赖的dylib。针对处理run-path-dependent libraries，dyld分为两个步骤完成：先提取所有包含的run-path搜索路径，然后再通过搜索列表里的路径来寻找和加载所有的run-path-dependent libraries。

为了提取所有的run-path搜索路径，dyld调用ImageLoader类的getRpaths()方法。该方法（被recursiveLoadLibraries()方法调用）简单的解析程序中所有的LC_RPATH加载命令。对应每个这种加载命令，dyld提取出run-path搜索路径并添加到一个向量中（例如：一个表），如图16所示。

![](http://drops.javaweb.org/uploads/images/a9e832e4ca2eda097f2aadfe6404dc45d020b507.jpg)

图16 提取并保存所有内置的run-path搜索路径

有了run-path搜索路径列表，dyld就可以找到所有依赖的run-path-dependent libraries了。这部分逻辑代码在dyld.cpp的loadPhase3()函数中。如图17所示，[[email protected]](http://drops.com:8000/cdn-cgi/l/email-protection)字。如果有，dyld就循环迭代run-path搜索表，替换‘@rpath’关键字，然后尝试从新生成的路径加载dylib。

![](http://drops.javaweb.org/uploads/images/42a6eb1e7b7313a97bc15726256182b3e6a63225.jpg)

图17 搜索run-path搜索目录，找寻有@rpath的dylib。

重要的一点是dyld搜索的路径顺序是确定的，是符合LC_RPATH加载命令的顺序的。如图17中显示的代码片段显示，搜索循环会不停寻找，直到找到目标dylib或者是所有的路径都搜索了。

图18，图解了搜索过程。可以看到dyld搜索了不同的run-path搜索路径，为了找到需要的run-path-denpendent dylib。注意在这个例子中，目标dylib是在第二个搜索目录中找到的。

![](http://drops.javaweb.org/uploads/images/20199786a571e48988efdb537d481e0f95e777d5.jpg)

图18 Dyld搜索多个run-path搜索目录

总结一下到此的发现：一个OS X系统是可以被劫持攻击的，只要任何程序存在以下的任意条件： 1，包含一个LC_LOAD_WEAK_DYLIB加载命令，但是相关的dylib并不存在。 2，同时包含一个LC_LOAD*_DYLIB加载命令指向一个run-path-denpendent library（'@rpath'）和多个LC_RPATH加载命令。并且run-path-denpendent library没有在第一个run-path搜索目录中。

本文的余下部分会先讲述一个完整的dylib劫持攻击，然后给出几个不同的攻击（驻留，加载时劫持，远程注入等），最后总结下如何防御此类攻击。

为了帮助读者更好的理解dylib劫持攻击，我们会尽量给出劫持攻击的细节，包括尝试攻击，遇到的错误，到最后的成功。有了这些知识的帮助，就可以更容易的理解自动攻击，攻击场景识别，和如何防御。

回顾之前描述的例子程序（'rPathApp.app'）。我们用来解释连接run-path-denpendent dylib的。这个程序将会是我们劫持攻击的目标。

dylib劫持攻击的对象只能是存在漏洞的程序（满足前面讲述的两个劫持条件之一的程序）。因为本例子程序（rPathApp.app）需要连接一个run-path-dependent dylib，它也许就满足上面第二个条件。最简单的检测方式就是开启加载器的debug logging功能，然后在命令行简单的运行该程序。为了开启这种logging，需要设置DYLD_PRINT_RPATH环境变量。这会让dyld打印出@rpath扩展情况和dy[[email protected]](http://drops.com:8000/cdn-cgi/l/email-protection)现漏洞（例如：第一个扩展指向了不存在的dylib）如图20所示。

![](http://drops.javaweb.org/uploads/images/9b70869f1bfec296fb8bcb91f391858f55d52c10.jpg)

图20 存在漏洞的测试程序rPathApp

图20展示了加载器第一次寻找目标dylib时，在指定位置没有发现。和图19中显示的一样，在这种情况下，攻击者可以部署一个恶意的dylib到刚才第一次搜索的路径，之后加载器会加载恶意库。

我们创建了一个简单的dylib来扮演恶意的劫持库。为了能够在加载时自动执行，该dylib实现了一个构造函数。该构造函数在dylib成功加载后会自动执行。这是一个很好的特性，因为一般的dylib代码不会执行，直到主程序调用它的某个导出函数。

![](http://drops.javaweb.org/uploads/images/789b761caa62fa5724ca0f42189acc8096935a32.jpg)

图21 一个dylib的构造函数将会自动执行

编译组建完成后，将dylib重命名，改成目标库的名字：rpathlib。接下来，创建需要的目录结构（Library/One/rpathLib.framework/Versions/A/）并将恶意的dylib拷贝进该目录。这就保证了无论何时程序启动，dyld在搜索run-path-denpendent dylib时会找到劫持dylib。

![](http://drops.javaweb.org/uploads/images/3505b0027a194246afd6a29f6185201545353bb8.jpg)

图22 恶意dylib被放置在第一个run-path搜索目录中

不幸的是，这一次劫持尝试失败了。程序意外的崩溃了。见图23。

![](http://drops.javaweb.org/uploads/images/d0654e35e307173d81124aa290e14798ba190d0e.jpg)

图23 成功解析路径，然后崩溃

虽然失败了，但是好消息就是，加载器找到了并尝试加载劫持dylib（看图23中的'RPATH successful expansion…'日志消息）。虽然程序崩溃了，但是加载器还是抛出了一条详细的异常信息。这条异常看起来是自解释的：劫持库的版本和要求的版本不同。重新研究加载器的源代码，找到了抛出这条异常的代码。见图24。

![](http://drops.javaweb.org/uploads/images/790ba76f901f115de25d2e616076a0af9f761a9a.jpg)

图24 Dyld提取和比较合适的版本号

可以看到，加载器会调用doGetLibraryInfo()方法从被加载库中的LC_ID_DYLIB加载命令中提取兼容和当前的版本号。提取出来的兼容版本号（'minVersion'）然后在和程序要求的版本进行对比。如果版本号太低，一个不兼容的异常就会被抛出。

解决此兼容问题也不难，只需要通过在Xcode中更新版本号，重新编译下就行。见图25。

![](http://drops.javaweb.org/uploads/images/30431bdbb7f3a6fafef1786e40317dd8c5ba9380.jpg)

图25 设置兼容和当前版本号

检查重新编译的劫持dylib的LC_ID_DYLIB加载命令。确认已经更新版本号。见图26

![](http://drops.javaweb.org/uploads/images/1056d3840b769d38d52adcf93b85829a6c813c78.jpg)

图26 兼容版本号和当前版本号

更新版本号后的劫持dylib又被拷贝进程序的第一个run-path搜索目录。重启漏洞程序，显示加载器找到了劫持dylib并且尝试加载。可是虽然现在dylib版本兼容。但是一个新的异常被抛出，程序又一次崩溃。见图27。

![](http://drops.javaweb.org/uploads/images/fab33cc5bbeb9bd457dc09d71eb4b2ae16405259.jpg)

图27 ‘符号没有找到’异常

又一次，异常给出的解释清晰说明了加载器为什么抛出异常。程序连接动态库的目的就是获得动态库导出的功能（比如:函数，对象等）。一旦被需求的dylib被加载进内存，加载器就会尝试解析（通过导出符号）依赖库试图导出的功能对象。如果功能对象未发现就会连接失败，连接进程就会终止，导致主程序崩溃。

有几种方法可以确保劫持dylib导出正确的符号表，这样才能完整的进行连接。一个简单的方法就是劫持dylib直接仿造目标dylib的导出信息。也许这样就可以成功了，看起来有点复杂并且不同的dylib各有特点（比如，攻击另外一个dylib需要另外的导出信息）。一个更优雅的办法是简单的让连接器去别的地方寻则它要求的符号。当然别的地方就是指合法的dylib。在这个场景中，劫持dylib将简单的扮演一个代理或者是一个‘re-exporter’dylib，加载器将会跟随它的重导出指令，没有连接错误会被抛出。

![](http://drops.javaweb.org/uploads/images/f87e7a2d8daa827d9f9f79de8583906a66a171bd.jpg)

图28 重导出合法的dylib

需要付出一些努力，才能让重导出的库完美工作。第一步是回到Xcode，添加多个链接的flags到劫持dylib项目。这些flags包括“-Xlinker '，' reexport_library '，然后还有到包含漏洞程序真正需要导出接口的目标库的路径。

![](http://drops.javaweb.org/uploads/images/8feb4e74d533ab094253c2586e1bdecab8235f49.jpg)

图29 要求的链接flag，来实现re-exporting

这些链接flags会生成一个内置的LC_REEXPORT_DYLIB加载命令。其中包含到目标dylib的路径。见图30。

![](http://drops.javaweb.org/uploads/images/9b4fd85a54fa156da41516175343071b1bbf9f53.jpg)

图30 内置的LC_REEXPORT_DYLIB加载命令

然而，事情并非如此简单。因为劫持dylib重导出目标是一个run-path-denpendent library。LC_REEXPORT_DYLIB（从合法的dylib的LC_ID_DYLIB加载命令中导出）中name段是以‘@rpath’开始的。这是有问题的，因为不像LC_LOAD*_DYLIB加载命令，dyld不会解析LC_REEXPORT_DYLIB加载命令中的run-path-denpendent路径。换句话说，加载器会直接加载‘@rpath[[email protected]](http://drops.com:8000/cdn-cgi/l/email-protection)的。

解决办法是移除内置‘@rpath’路径，提供目标库的完整路径给LC_REEXPORT_DYLIB加载命令。这需要借助一款苹果开发者工具：install_name_tool。来更新LC_REEXPORT_DYLIB加载命令中的install name。这个工具执行时，用-change选项，随后是现存的name（LC_REEXPORT_DYLIB中的），新的name，劫持dylib的路径。见图31。

![](http://drops.javaweb.org/uploads/images/5330716af68675f7df0a4b71ab27875e26ef257e.jpg)

图31 使用installl_tool_name来更新内置的name

在LC_REEXPORT_DYLIB加载命令正确更新后，劫持dylib被重新拷贝到主程序的第一个run-path搜索目录，重启程序。如图32，最终成功执行。

![](http://drops.javaweb.org/uploads/images/e5ab6792d956348646c9c0caf76e354e1a4222ea.jpg)

图32 成功劫持又漏洞的程序

总结一下：因为rPathApp程序连接一个run-path-denpendent库，而这个库在第一个run-path搜索目录中没有找到，所有存在了dylib劫持漏洞。植入一个特殊兼容的恶意dylib在第一个搜索目录中会导致在每一次程序执行时，加载器都会盲目的加载这个恶意dylib。因为这个恶意dylib拥有正确的版本信息，同时重导出了合法目标dylib的所有符号，所有需要的符号都能解决，因此保证了程序的功能不会受损。

0x03 参考
=====

1.  Secure loading of libraries to prevent DLL preloading attacks.[http://blogs.technet.com/cfs-file.ashx/__key/CommunityServer-Components-PostAttachments/00-03-35-14-21/Secure-loading-of-libraries-to-prevent-DLL-Preloading.docx](http://blogs.technet.com/cfs-file.ashx/__key/CommunityServer-Components-PostAttachments/00-03-35-14-21/Secure-loading-of-libraries-to-prevent-DLL-Preloading.docx).
2.  DLL hijacking.[http://en.wikipedia.org/wiki/Dynamic-link_library#DLL_hijacking](http://en.wikipedia.org/wiki/Dynamic-link_library#DLL_hijacking).
3.  Dynamic-Link Library Hijacking.[http://www.exploit-db.com/wp-content/themes/exploit/docs/31687.pdf](http://www.exploit-db.com/wp-content/themes/exploit/docs/31687.pdf).
4.  Windows NT Security Guidelines.[http://www.autistici.org/loa/pasky/NSAGuideV2.PDF](http://www.autistici.org/loa/pasky/NSAGuideV2.PDF).
5.  What the fxsst?[https://www.mandiant.com/blog/fxsst/](https://www.mandiant.com/blog/fxsst/).
6.  Leaked Carberp source code.[https://github.com/hzeroo/Carberp](https://github.com/hzeroo/Carberp).
7.  Windows 7 UAC whitelist: Proof-of-concept source code.[http://www.pretentiousname.com/misc/W7E_Source/win7_uac_poc_details.html](http://www.pretentiousname.com/misc/W7E_Source/win7_uac_poc_details.html).
8.  Microsoft Security Advisory 2269637; Insecure Library Loading Could Allow Remote Code Execution.[https://technet.microsoft.com/en-us/library/security/2269637.aspx](https://technet.microsoft.com/en-us/library/security/2269637.aspx).
9.  What is dll hijacking?[http://stackoverflow.com/a/3623571/3854841](http://stackoverflow.com/a/3623571/3854841).
10.  OS X loader (dyld) source code.[http://www.opensource.apple.com/source/dyld](http://www.opensource.apple.com/source/dyld).
11.  MachOView.[http://sourceforge.net/projects/machoview/](http://sourceforge.net/projects/machoview/).
12.  Run-Path Dependent Libraries.[https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/RunpathDependentLibraries.html](https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/RunpathDependentLibraries.html).
13.  Using @rpath: Why and How.[http://www.dribin.org/dave/blog/archives/2009/11/15/rpath/](http://www.dribin.org/dave/blog/archives/2009/11/15/rpath/).
14.  Friday Q&A 2012-11-09: dyld: Dynamic Linking On OS X.[https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html](https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html).
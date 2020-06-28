# HackingTeam源码泄漏——语音监控分析

author:猎豹安全中心

0x00 前言
=====

在HackingTeam泄漏的文件，我们发现了有针对主流聊天软件中的语言进行监控的代码，其中包括国内常用的微信。下面就以微信为例，来分析一下HackingTeam是如何实现语言监控的。

语音监控的相关代码在`core-android-audiocapture-master`文件夹下，通览全部源码之后，我们发现，语音监控的实现，主要是通过ptrace实现代码注入，将一个动态库注入到微信的进程中实现的。由于Android系统对权限的控制，要实现该目的，必须要拥有管理员权限才可以。也就是说，恶意软件需要先获取root权限，之后才能进一步实现语音监控。

0x01 实现
=====

下面进入主题，说说是如何实现语音监控的。

`core-android-audiocapture-master\dbi_release`文件夹里面是程序主要代码，包括的文件如下

![enter image description here](http://drops.javaweb.org/uploads/images/fe675f5553428eafa6529022df209856fce741d5.jpg)

其中，hijack文件夹下包含了一份进程注入的代码，该代码的主要功能就是通过ptrace将一个动态链接库(*.so文件)注入到指定的进程里面，并执行。实际上关于进程注入的代码，之前已经有过很多个版本了，关于细节就不再赘述。相信在不久之后，基于HackingTeam泄漏版进行改写的hijack将会大量的出现。

当动态链接库文件被注入到微信进程之后，会直接调用初始化函数，该函数即为libt.c中的my_init函数。

在my_init函数开始部分，首先对系统版本进行了判断，并根据不同的版本采取不同的措施。

![enter image description here](http://drops.javaweb.org/uploads/images/2c7ac6cf5160a72c296a4fc0ba8aeafa46bc4dfa.jpg)

为了更容易理解，我们选取android 4.0的环境，看看my_init都做了写什么事情。

![enter image description here](http://drops.javaweb.org/uploads/images/5dcefcff6a3d0c2fa2efc6ef5c406d57e3c2d628.jpg)

可以看到很多`HOOK_coverage_XX`形式的变量，这实际上是函数调用的宏定义。相关的定义在`hijack_func\hooker.h`文件中

![enter image description here](http://drops.javaweb.org/uploads/images/5a82c08f10e20a00d4ed1abf69c2b16dd9328c68.jpg)

根据上图可以知道，HOOK_coverage_XX实际上是调用了help_no_hash函数。那么我们先来总结一下help_no_hash函数都做了些什么，该函数在libt.c文件中定义。

首先，help_no_hash调用find_name来获取指定函数地址。find_name的实现在util.c文件中，过程就是通过指定的pid读取/proc//maps文件，获取libname的基址Base和文件路径Path，随后根据Path读取libname这个文件并解析，获取funcname的相对地址偏移VA，最后计算Base+VA就是funcname在进程中的实际地址，最后写入addr中。最后把libname所在的内存属性设置为可读写和执行，方便后期进行修改。

![enter image description here](http://drops.javaweb.org/uploads/images/e4884d17e8c2b5cc43a311d1e9bd6491cf157c77.jpg)

经过find_name的调用，就可以找到funcname函数的实际地址，就是addr的值。之后通过判断addr的值，就可以知道该函数指令集是ARM还是THUMB。并根据不同的指令集进行不同的处理。重点看一下ARM指令集的处理

![enter image description here](http://drops.javaweb.org/uploads/images/0756a07efa732a10c951d8257684ff1722d17cf2.jpg)

根据上图，可以看到，主要内容就是初始化hook变量，hook变量的类型是struct hook_t *，随后修改funcname函数的前12个字节，实现inline hook。

对于THUMB指令集，实现的功能也类似。都是inline hook，只是在具体指令上有些差别。此处就不再赘述了。

综合上述的信息，总结一下，help_no_hash函数的主要功能就是对于指定的函数实现inline hook。相信该部分代码以后也会流行起来的。但是该段代码虽然写的比较漂亮，但是依旧存在一些问题，诸位发挥拿来主意的时候，最好还是多测试一下。

回到主题，既然知道了help_no_hash是实现了inline hook功能，那么就来总结一下，都hook了哪些函数。

![enter image description here](http://drops.javaweb.org/uploads/images/dc078e20aef2430580e098c443ac51cbcaafdede.jpg)

上表总结了hook关系。我们重点关注的是”Hook函数”这一项，里面的内容就是实际的代码。各个函数主要是实现监控并记录的功能，我们挑选一个比较有代表性的”newTrack_h”来进行分析。

newTrack_h函数实现是在hijack_func\hooker_thumb.c文件中。

由于使用的是inline hook，并且目的是监控，而非控制，所以，在每个Hook函数开始位置都有相似的调用原始函数的过程。通过helper_precall和helper_postcall来实现安装和卸载inline hook的功能。

![enter image description here](http://drops.javaweb.org/uploads/images/a88b58ee6cb649aa67b8523bab541500532ba9f2.jpg)

之后经过一些无关紧要的操作，生成一个struct cblk_t *类型的结构体变量cblk_tmp，并进行初始化。

![enter image description here](http://drops.javaweb.org/uploads/images/1522a868dd9a0b04337391d3ac24badb54cb2bec.jpg)

然后为cblk_tmp->filename生成一个值，作为保存的文件名。

![enter image description here](http://drops.javaweb.org/uploads/images/485bcc70f36767b7c366497f9a575c7f75925ff6.jpg)

以cblk_tmp->filename作为文件名创建文件

![enter image description here](http://drops.javaweb.org/uploads/images/2d4ffb9317d23a4cffaff1c15703db5e9b41acb5.jpg)

继续对cblk_tmp的成员变量进行赋值，最后放入哈希表中，方便后续的查找等操作。

![enter image description here](http://drops.javaweb.org/uploads/images/940fd48e01502f7e67d373bead6649ae86aca876.jpg)

至此，newTrack_h函数的主要流程就描述完了。相信大家一定很奇怪，说好的语音监控呢？根本没有看到啊。事实上，用于记录语音的操作是在recordTrack_getNextBuffer3_h和playbackTrack_getNextBuffer3_h中实现的，从名字就能看出来，这两个函数分别用于在录音和播放的时候，用于记录音频内容。由于这两个函数比较复杂，我们只重点介绍一下音频记录相关的部分。

选取playbackTrack_getNextBuffer3_h进行分析，该函数实现在用户播放语音的时候，进行记录。

在函数的开始部分，依旧是调用原始的函数。

![enter image description here](http://drops.javaweb.org/uploads/images/4a6d3e286603e365a99097c53b3e1a4c043635e1.jpg)

之后，会根据系统版本，通过硬编码的偏移值，获取系统结构地址。

![enter image description here](http://drops.javaweb.org/uploads/images/f315a68ccbd7dfde060e8c79a3c8dec5e3f5a7c7.jpg)

然后将信息写入文件

![enter image description here](http://drops.javaweb.org/uploads/images/935d326e8198d464a3e4fbcfad696f31cebd5c4a.jpg)

注意到，bufferRaw就是音频的信息，根据之前的表格，可以看到playbackTrack_getNextBuffer3_h实际上是对AudioFlinger::PlaybackThread::Track::getNextBuffer函数进行的Hook，其参数AudioBufferProvider::Buffer *的内容，就是音频内容，也就是bufferRaw的来源。通过记录bufferRaw，就能记录原始的音频内容了。

![enter image description here](http://drops.javaweb.org/uploads/images/6abce007230a9256bdf3b15b9bdb12cca84df96b.jpg)

在recordTrack_getNextBuffer3_h函数中也有类似的操作，就不赘述了。

![enter image description here](http://drops.javaweb.org/uploads/images/a5f17a1ab6b57baa90a669fa758ba06ffa932a25.jpg)

从记录方式上来看，该代码按照自己定义的方式来记录信息的，生成的也是bin文件。所以生成的文件并不能直接用于播放。好在HackingTeam预留了一个转换文件，在decoder\ decoder.py文件中，实现了将记录的bin文件转换为wav文件的功能。

0x02 结语
=====

至此，语音监控功能的源码分析也就基本完成了。总体看来，HackingTeam的代码写的还是非常漂亮的，其中动态库注入部分，以及inline Hook部分源码，预计在未来一段时间会被借鉴与各种Android项目中。但是这分代码依旧存在一些问题，首先是必须root权限才能正确执行，而且在实现的过程中，使用了一些硬编码，而Android系统本身碎片化十分严重，各种定制ROM流行，这就使得硬编码只能适配少数一部分系统，而无法做到对所有Android系统都有效。相信这也是一种无奈之举吧。
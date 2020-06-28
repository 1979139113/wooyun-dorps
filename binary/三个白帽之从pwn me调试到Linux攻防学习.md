# 三个白帽之从pwn me调试到Linux攻防学习

0x00 写在前面
=====

从**z神@explorer**发表关于《来pwn我一下好吗》的write up已经过去一段时间了，这篇write up对于我这样的Linux漏洞攻防新手来说很多地方都是很难理解的，后来结合这篇write up，我又调试了一下这个pwn me的题目，发现无论是从error出的题目，还是z神写的write up，都特别精彩，有很多值得学习的地方。

这个漏洞writeup从各个角度看，涵盖了很多Linux下的攻防技巧，于是我把我分析思考的过程，进行了一个总结，拿出来跟大家一起分享，或者说，我把这个漏洞拨茧抽丝，提取一些最有意思的部分，来和大家一起享受这个精彩的pwn me之旅。

在开头的最后，必须再次感谢一下出题人**Error**以及 writeup的作者**Explorer**，真的学习到了很多东西。

这篇文章需要结合z神的writeup来看，传送门在此[http://drops.wooyun.org/binary/16257](http://drops.wooyun.org/binary/16257)

文中若有不到位或者描述有误之处，请大牛们批评指正，多多交流，也希望能对大家有所帮助！（抱拳！）

0x01 思考与问题
=====

在阅读这篇writeup的时候产生了很多疑问，其实主要还是对于Linux下堆管理机制，以及一些Linux攻防技巧不熟的原因导致的，其实漏洞并不难处理，关键在于如何大开脑洞来利用，下面我抛出我在做这个pwn me调试时产生的问题，并且通过动态调试的过程来解决这些疑问。

*   fastbin chunk如何导致任意地址写的？
*   libc基址到底是如何泄露的？
*   如何通过写入free hook来达到执行/bin/sh的？

这三个问题其实都只是引子，在动态调试的过程中，发现了很多有趣的地方。在开始调试前，我想再啰嗦一下为什么会产生这三个问题。

首先第一个问题，fastbin这种malloc机制比较好理解，单向链表，也就是说只有一个指向下一个chunk的指针，但是如何利用这个指针达到任意地址写的呢？

第二个问题，从z神的writeup利用中提到了两种leak方法，那么它们的工作机制是怎样的呢？

第三个问题，writeup中提到的通过改写free hook达到执行/bin/sh的目的，那么这个过程又是什么样的呢？

下面带着这三个问题，开始最精彩的旅行～

0x02 关于z神提到的几个坑
=====

在writeup中z神提到了几个坑，其中一个是check函数的，作者error后来也在文末提到了，两种方法，一种是利用任意地址写来修改全局变量a000的大小，另一种是直接无视掉，因为确实有时候会出现check函数检测没有通过的情况。

![](http://drops.javaweb.org/uploads/images/3939d81c74e3a3da78e66b9187f17f951b3466ff.jpg)

另外一个坑是关于FULL RELRO保护，其实通过gdb－peda可以直接查看保护开启情况。

![](http://drops.javaweb.org/uploads/images/6547bacee5d315f2923132f2942dfa0abc5d9b04.jpg)

这个保护主要是针对got表，这个got表存放着很多函数指针，针对got表攻防的概念有点像Windows下虚函数的攻击，关于got表的攻击有一篇文章讲的挺好的，传送门在此[http://drops.wooyun.org/binary/6521](http://drops.wooyun.org/binary/6521)

这里不再赘述，主要是z神在这个pwn题目中利用方法也很有意思。

0x03 从leak addr到任意地址写
=====

说到leak addr就要从fastbin chunk说起，fastbin chunk是Linux下堆的一种分配方式，比较简单，采用的是单向链表的方式，只有一个fd指针指向下一个未分配的chunk，利用这种方法，可以泄露出堆的地址，这和后面的任意地址写有很重要的关系。

对于fastbin chunk，网上的说明有很多了，其实简化版的图是这样的。

![](http://drops.javaweb.org/uploads/images/b5e5ca364ae4e9923322c12a117ec58bcc79c544.jpg)

在这个pwn题目中，对Vul结构的构造，rank恰好处于fd指针的位置，那么接下来在delete函数中调用free的时候，这个rank的位置会泄露堆地址，这是fastbin chunk的机制。

在这个漏洞中，要分别malloc申请两个fastbin chunk，然后对这两个chunk进行free操作，首先来看一下申请过程。

![](http://drops.javaweb.org/uploads/images/bfaaddab5ba71531f16d7ba72d9fdb34c04dd40c.jpg)

在6040a0地址位置是rank存放的位置，也就是fd的位置，下面要关注这个地址，其他chunk的申请过程我不再展示了，直接来看看free chunk之后构成fastbin chunk链表的过程。

下面对这个chunk执行free操作。

![](http://drops.javaweb.org/uploads/images/0efccc1bd57999fde6ad20d90ebc691c7918dbe5.jpg)

这里要提的一点是两个free，在pwn题目中，一个free是对detail指针的free，另一处是free整个memory，观察地址00604110位置，其实这里就是之前vul结构中detail指针malloc地址的位置，这里作为006040a0之前的chunk释放，而detail指针位置恰好是fd的位置，因此这里指向了00604070，也就是下一个未使用chunk的地址。

接下来观察eip处于memory free的函数调用处，接下来单步步过，观察006040a0地址。

![](http://drops.javaweb.org/uploads/images/d71bb34c190cdeb8cb68bef459c79e89eb80fc06.jpg)

可以看到，这时006040a0位置已经泄露了堆地址，而这个位置正好之前提到的rank的地址，而在writeup中已经提到了，在rank赋值时有一处if语句判断，如果rank大于0则赋值，因此，只要控制rank小于等于0，则不会赋值，通过这种方法，再通过程序的show函数，就可以通过rank打印出堆地址。

**那么其实这个过程，就是在二进制漏洞攻防中，常用于bypass ASLR的利用方式，而上述的这个调试情景，就是一个最简单的过程**

那么接下来，又是如何达到任意地址写的过程的呢？这就要提到pwn题目中edit函数存在的use after free释放后重利用，这个其实是uaf中一个非常简单的情况，在chunk释放后，没有对其进行标记，从而导致在释放后仍然能对该chunk进行操作，通过这种方式，可以修改fd的值，这就是一个典型的fake chunk的利用场景了，这样如果再次malloc，可以讲地址跳到任意地址，从而达到任意地址写的目的。

关于fake chunk，freebuf上有一篇不错的文章，传送门在此[http://www.freebuf.com/news/88660.html](http://www.freebuf.com/news/88660.html)

接下来我们来看一下这个uaf漏洞是如何通过uaf漏洞造成fake chunk的，这里我要说一下，在调试的过程中，虚拟机很不给力的挂了两次，导致调试时堆地址发生了改变，但整个过程和描述是不变的。（忘了用快照的悲剧。。。）

首先是free之后在edit中可以对rank进行赋值，而此时实际上rank所处的chunk已经被释放了。

![](http://drops.javaweb.org/uploads/images/b41b87835ca50fd7759a5a5bae4ce190ccc2c587.jpg)

可以看到，尽管该chunk已经被释放，但是title标记仍然存在，也就是说，在memcmp的过程中仍然可以匹配到title的值，从而可以继续接下来的赋值，就是这样的过程，令我门可以控制rank的位置，而之前也提到这里可以创造fake chunk，接下来重新申请。

![](http://drops.javaweb.org/uploads/images/6c8d7b324099cfae8f26b965f7b06a7eb0966b3e.jpg)

这里地址改变的原因之前已经提到，在这里的fake chunk地址是我随便输入的，证明fake chunk在fastbin chunk申请过程中对chunk地址的影响。

![](http://drops.javaweb.org/uploads/images/473ced9dd817b7dbc785c2e72c35569ac460babf.jpg)

0x04 fast bin与normal bin中的leak libc
=====

在开头的三个问题中，我提到了一个泄露libc基址的问题，其实在上面一个小节中，已经提到了泄露堆基址的方法，以及任意内存读写的方法，那么其实在z神的writeup中，提到了leak libc的两种方法，第一种是他在文中提到的泄露libc基址的方法，在我调试的过程中，确实发现了在chunk内某个偏移位置存在一个libc的地址，那么利用任意地址读的方法就可以找到这个libc的地址，以此计算出libc的基址。

![](http://drops.javaweb.org/uploads/images/0111dee709bd8b2871173a6939a80021f0f98044.jpg)

其实这里我想说的是z神提到的另一种方法normal bin，normal bin和fast bin差别比较大，首先，normal bin chunk采用的是双向链表，在后面动态调试中可以看到，而在normal bin双向链表表头地址是在libc中，当时我的疑问也是在于，**释放chunk后会产生libc的指针到底是怎么一回事**。

带着这个疑问来调试这个原理，位置仍然是在add函数中，首先对vul结构第一个malloc位置没法控制，但是对于detail指针却是可以控制的，通过控制detail长度，我们就可以让fastbin chunk变成normalbin chunk，我们申请一个大小为4000的chunk，这时候，linux就会采用normal bin的机制。

![](http://drops.javaweb.org/uploads/images/5c37440d20c8a10cd74eecf5dd012d85903fe4a5.jpg)

我们需要申请两个chunk，在释放时构成双向链表，在第一个chunk释放的时候，chunk空间并没有什么变化。

![](http://drops.javaweb.org/uploads/images/301d3fe92d942807253fc6f8e8ae56160cb11369.jpg)

接下来对第二个chunk进行释放，首先到达free的位置前，来观察一下chunk空间的情况。

![](http://drops.javaweb.org/uploads/images/3a7d05e2decbc8bfd3c39f9554e4e931addd3187.jpg)

单步步过，在此观察chunk中的情况。

![](http://drops.javaweb.org/uploads/images/7d1367ec61ffa496bac42e63eef8e82e0ca4ef7e.jpg)

可以看到，此时位于libc中的链表头部指针已经暴露出来了，可以再通过任意地址读取的方法获得到这libc中的地址，之后可以计算出libc的基址地址。

0x05 覆盖free执行system
=====

在got表被保护的情况下，通过任意地址写的方法，可以修改关键地址，从而达到执行代码的目的，刚开始一直对z神 write up中关于最后执行的地方百思不得其解，后来通过调试终于发现这到底是怎么回事了。

其实在前面的小节我已经提到过，在free的时候，会先free detail指针，而detail指针在创建的时候是可控的，于是按照z神的说法，在detail指针写入/bin/sh，这样在free detail的时候，如果通过任意地址写覆盖掉free，将system函数的地址覆盖上，这样free detail就会变成system detail，于是最后delete函数关键位置变成执行：**system(/bin/sh)**

这样就可以在目标端口拿到shell了（好像比shellcode还方便，但真实攻防中很难）

![](http://drops.javaweb.org/uploads/images/ffb2127559c5276612c39340e9944a27b5634638.jpg)

0x06 二进制学习的一些心得
=====

到这里，所有疑问都基本上解决了，感觉学到了很多东西，我接触二进制的时间比较短，还在努力摸索，这里想分享一些小心得，希望大家能多多指点。

*   第一个是多调试，很多东西光看很难理解，但是如果自己去调试的话很快就能理解。
*   第二个是把复杂问题简单化，其实这么表述也不一定正确，我的意思其实就是把一些复杂的逻辑或者漏洞原理转换成一个简单的模式，比如一个demo，这样能更快的熟悉这个漏洞最本质的原理。

最后感谢三个白帽，ca叔邀请我到三个白帽做这个pwn题的时候还是看到我离大牛们还是有不少的差距，今后还是希望自己能够多受点挫折，多进步一点！
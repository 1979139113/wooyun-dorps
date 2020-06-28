# 浏览器fuzz框架介绍

本文简要介绍了流行的三个浏览器动态Fuzz工具cross_fuzz、grinder、X-Fuzzer的原理及其优缺点，并提供了一种通过动静结合的方式兼顾可重现性、通用性、高效性、自动化程度高的浏览器fuzz方法。

0x00 引言
=====

Fuzz（模糊测试）是一种侧重于发现软件安全漏洞的方法。典型地模糊测试过程是通过自动的或半自动的方法，反复驱动目标软件运行，并为其提供”精心”构造的输入数据，同时监控软件运行的异常，进而根据异常结果及输入数据查找软件的安全漏洞。随着Smart Fuzz的发展，RCE（逆向代码工程）需求的增加，其特征更符合一种灰盒测试。其简要流程图如下：

![](http://drops.javaweb.org/uploads/images/b6982fd4746eeb95b52be9368bf76eb21281ac04.jpg)

Web浏览器是网络应用中使用最广泛的软件之一，IE、FireFox、Chrome等三款主流浏览器占据了Web浏览器市场的大部分份额，其自身的安全性备受关注、影响广泛。本文介绍的浏览器fuzz则是以上述三款浏览器作为fuzz的主要目标，以内容（css、html、js）随机的html页面作为被测浏览器的输入，并监控浏览器运行的异常情况，查找其安全漏洞。

浏览器fuzz是查找浏览器漏洞的一个常用且有效的方法，公开的动态浏览器fuzz工具有cross_fuzz、grinder、X-Fuzzer等，这些工具基本都通过javascript脚本进行fuzz操作，同时通过hook函数、localStorage本地存储等技术手段动态记录fuzz操作日志，捕获到异常后再根据记录日志进行还原。

现有浏览器fuzz工具动态记录日志方式可能会影响被测浏览器的执行环境进而导致有时不能够重现异常；且其在记录日志方法上通用性（如IE的ActiveXObject）、稳定性（grinder hook函数）欠佳。

很多浏览器fuzz工具每运行一个测试用例都会重启待测浏览器进程，因为浏览器重启在运行一个测试用例过程中耗时占比较大，进而导致fuzz效率不高。

有的浏览器fuzz工具（如经典的cross_fuzz）在遇到crash时会停止fuzz，自动化程度不够。

本文在总结了现有动态浏览器fuzz工具优缺点的同时，提供了一种通过动静结合的方式兼顾可重现性、通用性、高效性、自动化程度高的浏览器fuzz方法及其工具NBFuzz（New Browser Fuzz）。

0x01 corss_fuzz 简介
=====

cross_fuzz由google的安全研究人员Michal Zalewski开发，支持多个浏览器fuzz，并专门针对IE浏览器作了优化。这个工具及在其基础上衍生的浏览器fuzz工具发现了大量的浏览器安全漏洞，其设计思想对浏览器漏洞挖掘产生了深远的影响。

cross_fuzz主要阐述了浏览器fuzz的设计思想，只是个演示性的功能模块，还不是一个完整的浏览器自动化fuzz工具，如关键的操作日志记录、异常监控等还需要用户自己来实现。

界面
--

![](http://drops.javaweb.org/uploads/images/a6237cac49a5f274d7b4c277236ccdb26e6ad0cf.jpg)

流程图
---

![](http://drops.javaweb.org/uploads/images/c65fd8b99f96b6fa4d1af60ca1cff2aa56c37dd2.jpg)

0x02 X-Fuzzer 简介
=====

X-Fuzzer是由安全研究人员Vinay Katoch开发的一款轻量级的动态浏览器fuzz工具，其日志记录采用了主流浏览器通用localStorage本地存储、document.cookie，即使浏览器异常崩溃时日志也能够保存。此工具dom元素fuzz操作、日志记录都很简单，还需自行完善；而且此工具没有异常监控模块，不能实现自动化fuzz。

界面
--

![](http://drops.javaweb.org/uploads/images/fb3719e8a1c8ba5385fda8bfc4e1eeacdeb74556.jpg)

功能模块图
-----

![](http://drops.javaweb.org/uploads/images/870271ef0cd69aa8d164e61e71503fd080a488fd.jpg)

0x03 grinder 简介
=====

grinder是一个自动化浏览器fuzz框架，客户端node主要采用ruby语言编写。

其日志记录通过向被测浏览器进程注入grinder_logger.dll ，进而hook javascript函数parseFloat在jscript9.dll、mozjs.dll等脚本引擎中的实现函数，这样需要记录日志时只需调用parseFloat函数，日志记录功能由grinder_logger.dll中相应的hook回调函数完成。需要注意的是下载相应dll符号文件后才能完成hook操作，且最近版本的Firefox已不包含mozjs.dll导致hook函数失败。此日志记录方法稳定性、通用性欠佳。

监控模块（ dbghelp.dll 、 symsrv.dll ）负责启动、监控被测浏览器，记录其异常信息，完成fuzz自动化 。

重现模块（ testcase.rb ）根据日志重现POC。

具体fuzz操作需要用户自行完善。

界面
--

![](http://drops.javaweb.org/uploads/images/26beb653a50d955079a52663329c6088defd7d52.jpg)

0x04 NBFuzz
=====

**NBFuzz 日志记录方法总结**

*   ActiveXObject只适用于IE。
*   cookie、html5的localStorage适合IE、Firefox、Chrome等主流浏览器，但存储大小有限制（cookie 4K、localStorage 5M），且需要设置浏览器支持cookie、记录历史。IE本地打开html文件时不支持localStorage。
*   html5本地数据库indexDB、SQLite的通用性不足（IE不支持或部分支持）且使用较复杂。
*   XMLHttpRequest通用性较好，但每一步fuzz操作都需要实时记录，通信总耗时较长，影响fuzz效率。
*   XMLHttpRequest改进：fuzz操作localStorage本地实时存储；每个测试用例开始时localStorage设置异常标志，没有异常在测试用例结束时置空异常标志；下一个测试用例通过XMLHttpRequest将产生异常的fuzz记录上传服务器。这样就解决了C/S实时记录fuzz操作日志的通信瓶颈且localStorage 大小能够满足要求。
*   抛弃fuzz操作日志：动态记录日志可能影响重现性（如indexDB 、 localStorage 等操作），通用性、稳定性（如grinder hook函数）欠佳；NBFuzz直接生成测试用例，不记录fuzz操作日志，且生成的测试用例不存在随机操作，保证了可重现性、通用性。

NBFuzz模块图
---------

![](http://drops.javaweb.org/uploads/images/d7b794a13753b8f0951f26e5a6be6a86ac1d32e0.jpg)

通过测试用例生成程序生成大量指定浏览器的fuzz测试用例，将生成的测试用例和调度页面一并置于web服务端，监控模块打开待测浏览器访问调度页面，调度页面通过内嵌iframe页面顺序调用测试用例；监控模块记录被测浏览器异常或者超时重启被测浏览器进程，保障浏览器fuzz的自动化；最后分析监控模块记录的异常信息，找出可疑crash，根据其异常时间查找web服务端log文件中修改时间与之相匹配的文件，获取调度页面发来的导致异常的测试用例名称，进而重现crash，进一步判断漏洞的可利用性。

测试用例生成
------

*   **界面**
    
    ![](http://drops.javaweb.org/uploads/images/1cd1d4e86250f84456badb0a1b9f9455915c6536.jpg)
    
    根据选定的浏览器类型生成大量包含其特性（不同的属性、函数、垃圾回收机制等）的测试用例，由于使用了IE特有的ActiveXObject进行本地文件操作所以要求使用IE运行此程序，使用js写包含js的测试用例。
    
*   **流程图**
    
    ![](http://drops.javaweb.org/uploads/images/c3908d4570c1a619ebe0dcf027d90f7e019067e9.jpg)
    

测试用例加载（调度模块）
------------

为解决每一个测试用例重启浏览器进程浏览器启动耗时占比大的问题，NBFuzz的调度模块通过内嵌iframe打开测试用例，一个测试用例完成后调度页面reload操作执行下一个测试用例，为防止页面不响应异常监控模块在超时后重启待测浏览器；同时为了防止web服务端记录log文件过多加入了不够精确的异常判断机制，因为超时重启将导致异常标志不能正常置false，产生误报，尽管如此也会大大减少log日志。

调度模块流程图如下：

![](http://drops.javaweb.org/uploads/images/76fcbe08c3e924f75f9386450b9b592690340ec3.jpg)

0x05 浏览器Fuzz总结
=====

随着浏览器的安全机制（如IE的延迟释放、隔离堆等）不断加强，其安全漏洞井喷之势已经得到遏制；而目前仍应用广泛的flash由于其安全机制的不健全逐渐成为安全研究的热点，安全漏洞亦呈现爆发的态势，可以预见不久的将来若其仍固步自封，必将步java的后尘，逐渐被用户所”抛弃”。

flash fuzz框架只需在NBFuzz框架的基础上进行些许的改变即可，如需增加as3测试用例编译成swf的过程。
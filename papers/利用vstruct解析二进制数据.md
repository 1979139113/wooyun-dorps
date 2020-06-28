# 利用vstruct解析二进制数据

原文地址：[http://williballenthin.com/blog/2015/09/08/parsing-binary-data-with-%60vstruct%60/](http://williballenthin.com/blog/2015/09/08/parsing-binary-data-with-%60vstruct%60/)

Vstruct是一个纯粹由Python语言编写的模块，可用于二进制数据的解析和序列化处理。实际上，Vstruct是隶属于vivisect项目的一个子模块，该项目是由[Invisig0th Kenshoto](http://visi.kenshoto.com/viki/MainPage)发起的，专门用来处理二进制分析。 Vstruct的开发和测试已经有许多年头了，并且已经集成到了许多生成环境下的系统中了。此外，这个模块不仅简单易学，而且重要的是，它还非常有趣！

您还在使用struct模块火急火燎地手工编写脚本吗？太苦逼了，不如使用vstruct吧！利用vstruct开发的代码，往往更具有陈述性或声明性，更加简明易懂，这是因为在编写二进制解析代码时通常会带有大量样板代码，而vstruct却不会出现这种情况。声明性代码强调的是二进制分析的下列重要方面：偏移，大小和类型。这使得基于vstruct的解析器更易于长期维护。

0x00 安装vstruct
=====

[Vstruct](https://github.com/vivisect/vivisect/tree/master/vstruct)模块是[vivisect](https://github.com/vivisect/vivisect)项目的一个组成部分，目前该项目与Python 2.7保持兼容，当然，面向Python 3.x的vivisect分支目前正在开发之中。

由于vivisect的子项目不是用兼容setuptools的setup.py文件分发的，所以你需要自己下载vstruct的源代码目录，并将其放入你的Python路径目录中，比如当前目录下：

```
$ git clone https://github.com/vivisect/vivisect.git vivisect
$ cd vivisect
$ python
In [1]: import vstruct
In [2]: vstruct.isVstructType(str)
Out[2]: False    

```

当然，通过setup.py来声明vstruct依赖的Python模块是非常麻烦的事情，因此为方便起见，我提供了一个[PyPI](https://pypi.python.org/pypi)镜像包，名为vivisect-vstruct-wb，这样的话，大家就可以直接利用pip命令来安装vstruct了：

```
$ mkdir /tmp/env
$ virtualenv -p python2 /tmp/env
$ /tmp/env/bin/pip install vivisect-vstruct-wb
$ /tmp/env/bin/python
In [1]: import vstruct
In [2]: vstruct.isVstructType(str)
Out[2]: False    

```

我已经对这个镜像进行了更新，现在它既支持Python 2.7也支持Python 3.0的解释程序，以便于读者在将来的工程中继续使用vivisect-vstruct-wb。另外，遇到问题时，千万不要忘了到Visi的GitHub上去看看有没有现成的答案。

0x01 Vstruct入门
=====

下面的例子相当于大家学编程语言时的“Hello World !”程序，它使用vstruct来解析字节串中的小端模式的32位无符号整数：

```
In [1]: import vstruct
In [2]: u32 = vstruct.primitives.v_uint32()
In [3]: u32.vsParse(b"\x01\x02\x03\x04")
In [4]: hex(u32)
Out[4]: '0x4030201'    

```

请注意观察上面代码是如何创建v_uint32类型实例、如何使用.vsParse()方法解析字节串，以及如何像处理原生Python类型实例那样来处理最后的结果的。为了更安全起见，我要显式地将解析后的对象转换成一个纯Python类型：

```
In [5]: type(u32)
Out[5]: vstruct.primitives.v_uint32    

In [6]: python_u32 = int(u32)    

In [7]: type(python_u32)
Out[7]: int    

In [8]: hex(python_u32)
Out[8]: '0x4030201'    

```

事实上，每个vstruct操作都被定义为一个以vs为前缀的方法，几乎在所有由vstruct派生的解析器中，都能找到这些方法的身影。虽然我最常用的是.vsParse()和.vsSetLength()这两个方法，但是我们最好熟悉所有方法的使用方法。下面是对每种方法的简单总结：

*   .vsParse()——从字节串解析实例。
*   .vsParseFd()——从文件类型的对象中解析实例（必然用到.read()方法）。
*   .vsEmit()——将实例序列化为字节串。
*   .vsSetValue()——利用原生Python实例为实例赋值。
*   .vsGetValue()——复制实例的数据，并将其作为原生Python实例。
*   .vsSetLength()——设置数组类型如v_str的长度。
*   .vsIsPrim()——如果实例为简单的primitive类型，则返回True。
*   .vsGetTypeName()——取得存放实例类型名称的字符串。
*   .vsGetEnum()——取得v_number实例关联的v_enum实例，如果存在的话。
*   .vsSetMeta()——（内部方法）。
*   .vsCalculate()——（内部方法）。
*   .vsGetMeta()——（内部方法）。

目前为止，vstruct看上去就像是struct.unpack的转基因克隆，所以，接下来我们有必要介绍它更酷的功能。

0x02 Vstructs的高级特性
=====

Vstruct解析器通常是基于类的。这个模块提供了一组基本数据类型（例如v_uint32和v_wstr分别用于DWORD和宽字符串），以及一个相应的机制来将这些类型组合成更加高级的数据类型（VStructs）。首先，我们先来介绍基本的数据类型：

*   Vstruct.primitives.v_int8——有符号整数。
*   vstruct.primitives.v_int16
*   vstruct.primitives.v_int24
*   vstruct.primitives.v_int32
*   vstruct.primitives.v_int64
*   Vstruct.primitives.v_uint8 -无符号整数。
*   vstruct.primitives.v_uint16
*   vstruct.primitives.v_uint24
*   vstruct.primitives.v_uint32
*   vstruct.primitives.v_uint64
*   vstruct.primitives.long
*   vstruct.primitives.v_float
*   vstruct.primitives.v_double
*   vstruct.primitives.v_ptr
*   vstruct.primitives.v_ptr32
*   vstruct.primitives.v_ptr64
*   vstruct.primitives.v_size_t
*   Vstruct.primitives.v_bytes——有确定长度的原始字节序列。
*   Vstruct.primitives.v_str——有明确长度的ASCII字符串。
*   Vstruct.primitives.v_wstr——有明确长度的宽字符串。
*   Vstruct.primitives.v_zstr——以NULL为终止符的ASCII码字符串。
*   Vstruct.primitives.v_zwstr——以NULL为终止符的宽字符串。
*   vstruct.primitives.GUID
*   Vstruct.primitives.v_enum——用来说明整数类型。
*   Vstruct.primitives.v_bitmask——用来说明整数类型。

复杂的解析器可以通过定义vstruct.VStruct类的子类来开发，因为vstruct.VStruct类可以包含众多变量，而这些变量可以是vstruct基本类型或高级类型的实例。好吧，我承认这句话有点绕口，那就一点一点来逐步消化吧！

```
    Complex parsers are developed by defining subclasses of the `vstruct.VStruct`
    class…    

class IMAGE_NT_HEADERS(vstruct.VStruct):
    def __init__(self):
        vstruct.VStruct.__init__(self)    

```

[源代码](https://github.com/vivisect/vivisect/blob/master/vstruct/defs/pe.py#L130)

在这个例子中，我们使用vstruct定义了一个Windows可执行文件的PE头部。我们的解析器名为IMAGE_NT_HEADERS，它是从类vstruct.VStruct那里派生出来的。我们必须在**init**()方法中显式调用父类的构造函数，具体形式可以是vstruct.VStruct.**init**(self)或者super(IMAGE_NT_HEADERS, self).**init**()。

```
    …that contain member variables that are instances of `vstruct` primitives…    

class IMAGE_NT_HEADERS(vstruct.VStruct):
    def __init__(self):
        vstruct.VStruct.__init__(self)
        self.Signature      = vstruct.pimitives.v_bytes(size=4)

```

[源代码](https://github.com/vivisect/vivisect/blob/master/vstruct/defs/pe.py#L130)

IMAGE_NT_HEADERS实例的第一个成员变量是一个v_bytes实例，它可以存放4字节内容。v_bytes通常用来存放无需进一步解析的原始字节序列。在本例中，成员变量.Signature的作用是，在解析有效PE文件时存放魔法序列“PE\x00\x00”。

在定义这个类的时候，还可以添加其他的成员变量，以用于解析二进制数据中不同部分的序列。类VStruct会记录成员变量的声明顺序，并处理其他相关的记录工作。唯一需要你去做的事情就是决定以哪种顺序来使用这些类型。够简单吧！

当某种结构在各种子结构中都要用到的时候，你可以将它们抽象成可重用的Vstruct类型，之后就可以像使用vstruct基本类型那样来使用它们了。

```
    [Complex parsers are developed by defining classes that contain] other complex `VStruct` types.    

class IMAGE_NT_HEADERS(vstruct.VStruct):
    def __init__(self):
        vstruct.VStruct.__init__(self)
        self.Signature      = v_bytes(size=4)
        self.FileHeader     = IMAGE_FILE_HEADER()    

```

[源代码](https://github.com/vivisect/vivisect/blob/master/vstruct/defs/pe.py#L130)

当Vstruct实例解析二进制数据遇到复杂的成员变量时，可以通过递归方式用子解析器来解决。在本例中，成员变量.FileHeader就是一种复合的类型，其定义见[这里](https://github.com/vivisect/vivisect/blob/master/vstruct/defs/pe.py#L80)。IMAGE_NT_HEADERS解析器首先会遇到.Signature字段的四个字节，然后，它把解析控制权传递给复合解析器IMAGE_FILE_HEADER。我们需要检查这个类的定义，以便确定其大小和布局情况。

我的建议是，开发多个Vstruct类，每个类负责文件格式的一小部分，然后使用一个更高级别的VStruct将它们组合起来。这样做的话，调试起来会更加容易一些，因为解析器的每一部分都可以单独进行检验。无论用什么方法，一旦定义好了一个Vstruct，你就可以通过文档开头部分描述的模式来解析数据了。

```
In [9]:
with open("kernel32.dll", "rb") as f:
    bytez = f.read()    

In [10]: hexdump.hexdump(bytez[0xf8:0x110])
Out[10]:
00000000: 50 45 00 00 4C 01 06 00  62 67 7D 53 00 00 00 00  PE..L...bg}S....
00000010: 00 00 00 00 E0 00 0E 21                           .......!    

In [11]: pe_header = IMAGE_NT_HEADERS()    

In [12]: pe_header.vsParse(bytez[0xf8:0x110])    

In [13]: pe_header.Signature
Out[13]: b'PE\x00\x00'    

In [14]: pe_header.FileHeader.Machine
Out[14]: 332    

```

在执行第9条命令的时候，我们打开了一个PE样本文件，并将其内容读入到了一个字节串中。在执行第10条命令的时候，我们用十六进制的形式展示了PE头部开头部分的一些内容。在执行第11条命令的时候，我们创建了一IMAGE_NT_HEADERS类的实例，但是需要注意的是，它还没有包含任何解析过的数据。此后，我们利用第12条命令显式解析了一个存放PE头部的字节串。通过第13和14条命令，我们展示了解析实例的成员的内容。需要注意的是，当我们访问一个嵌入的复合Vstruct时，我们可以继续进一步索引其内部内容，但是当我们访问一个基本类型成员时，我们得到的是原生Python的数据形式。说句实在话，这真是太方便了！

在进行调试的时候，我们可以通过.tree()方法把被解析数据以人类可读的形式打印出来：

```
In [15]: print(pe_header.tree())
Out[15]:
00000000 (24) IMAGE_NT_HEADERS: IMAGE_NT_HEADERS
00000000 (04)   Signature: 50450000
00000004 (20)   FileHeader: IMAGE_FILE_HEADER
00000004 (02)     Machine: 0x0000014c (332)
00000006 (02)     NumberOfSections: 0x00000006 (6)
00000008 (04)     TimeDateStamp: 0x537d6762 (1400727394)
0000000c (04)     PointerToSymbolTable: 0x00000000 (0)
00000010 (04)     NumberOfSymbols: 0x00000000 (0)
00000014 (02)     SizeOfOptionalHeader: 0x000000e0 (224)
00000016 (02)     Characteristics: 0x0000210e (8462)    

```

0x03 Vstruct高级主题
=====

**条件性的成员**

由于Vstruct的布局是在该类型的**init**()构造函数中定义的，所以，它能够对这些参数进行交互，并能够选择性的包含某些成员。举例来说，一个Vstruct在32位平台和64位平台上可以有不同的行为，如下所示：

```
class FooHeader(vstruct.VStruct):
    def __init__(self, bitness=32):
        super(FooHeader, self).__init__(self)
        if bitness == 32:
            self.data_pointer = v_ptr32()
        elif bitness == 64:
            self.data_pointer = v_ptr64()
        else:
            raise RuntimeError("invalid bitness: {:d}".format(bitness))    

```

这是一种非常强大的技术，尽管要想正确使用需要一点点小技巧。重要的是要了解它们的布局是何时最终确定下来的，何时用于估计，何时用于二进制数据的解析。当**init**()被调用时，这个实例并不会访问待解析的数据。只有当.vsParse()被调用时，成员变量中才会填上待解析的数据。因此，VStruct构造函数无法通过引用成员实例的内容来决定如何继续下面的解析工作。举例来说，下面的代码是行不通的：

```
class BazDataRegion(vstruct.VStruct):
    def __init__(self):
        super(BazDataRegion, self).__init__()
        self.data_size = v_uint32()    

        # NO! self.data_size doesn't contain anything yet!!!
        self.data_data = v_bytes(size=self.data_size)    

```

**回调函数**

为了正确地处理动态解析器，我们需要使用vstruct的回调函数。当一个VStruct实例完成了一个成员区段的解析时，它会检查这个类是否具有一个前缀为pcb_（解析器的回调函数）的同名方法，如果有的话，就会调用这个方法。同时，这个方法名称的其他部分就是刚才解析的区段的名称；举例来说，一旦BazDataRegion.data_size被解析完，名为BazDataRegion.pcb_data_size的方法就会被调用，当然，前提是这个方法确实存在。

这一点非常重要，因为当回调函数被调用时，VStruct实例已经被待解析的数据填充了一部分了，举例来说：

```
In [16]:
class BlipBlop(vstruct.VStruct):
    def __init__(self):
        super(BlipBlop, self).__init__()
        self.aaa = v_uint32()
        self.bbb = v_uint32()
        self.ccc = v_uint32()    

    def pcb_aaa(self):
        print("pcb_aaa: aaa: %s\n" % hex(self.aaa))    

    def pcb_bbb(self):
        print("pcb_bbb: aaa: %s"   % hex(self.aaa))
        print("pcb_bbb: bbb: %s\n" % hex(self.bbb))    

    def pcb_ccc(self):
        print("pcb_ccc: aaa: %s"   % hex(self.aaa))
        print("pcb_ccc: bbb: %s"   % hex(self.bbb))
        print("pcb_ccc: ccc: %s\n" % hex(self.ccc))    

In [17]: bb = BlipBlop()    

In [18]: bb.vsParse(b"AAAABBBBCCCC")
Out[18]:
pcb_aaa: aaa: 0x41414141    

pcb_bbb: aaa: 0x41414141
pcb_bbb: bbb: 0x42424242    

pcb_ccc: aaa: 0x41414141
pcb_ccc: bbb: 0x42424242
pcb_ccc: ccc: 0x43434343    

```

这就意味着，我们可以推迟一个类的布局的最终初始化，直到某些二进制数据解析完成为止。下面是实现一个规定大小的缓冲区的正确方法：

```
In [19]:
class BazDataRegion2(vstruct.VStruct):
    def __init__(self):
        super(BazDataRegion2, self).__init__()
        self.data_size = v_uint32()
        self.data_data = v_bytes(size=0)    

    def pcb_data_size(self):
        self["data_data"].vsSetLength(self.data_size)    

In [20]: bdr = BazDataRegion2()    

In [21]: bdr.vsParse(b"\x02\x00\x00\x00\x99\x88\x77\x66\x55\x44\x33\x22\x11")    

In [22]: print(bdr.tree())
Out[22]:
00000000 (06) BazDataRegion2: BazDataRegion2
00000000 (04)   data_size: 0x00000002 (2)
00000004 (02)   data_data: 9988    

```

在第19条命令中，我们声明了一个结构，它具有一个头字段（.data_size），指示随后的原始数据(.data_data )的大小。因为当时我们还没有这个待解析的头部的值。之后，**init**()被调用，我们使用了一个名为.pcb_data_size()的回调函数，它将在解析.data_size区段时被调用。当这个回调函数执行时，会更新.data_data字节数组的大小，以便使用正确的字节数量。在执行第20条命令的时候，我们创建了一个解析器的实例，然后，利用第21条命令对一个字符串进行了解析处理。虽然我们传入了13个字节，但是我们希望只用其中的6个字节：4字节用于uint32型变量.data_size，2字节用于字节数组.data_data。而其余的字节则不做处理。在执行第22条命令的时候，结果表明我们的解析器对二进制数据进行了正确的解析。

请注意，在执行回调函数.pcb_data_size()期间，我们使用方括号访问了Vstruct实例中名为.data_data的对象。之所以这样做，是因为我们既想要修改子实例本身，但是又不想从子实例中取得待解析的具体值的缘故。要想弄清楚到底应该使用哪种技术 (self.field0.xyz 或 self["field0"].xyz)，需要读者自己在实践中摸索一下，但通常来说，如果你想要解析一个具体值的话，就应该避免使用方括号。

0x04 小结
=====

在我们开发可维护的二进制代码解析器的时候，vstruct模块是一个强大的助手。它能够去除开发过程带来的大量样本代码。我特别喜欢使用vstruct解析恶意软件的C2协议、数据库索引和二进制的XML文件。如果读者感兴趣的话，我建议大家通过下列项目来学习vstruct解析器：

*   Vstruct的定义，地址为`https://github.com/vivisect/vivisect/tree/master/vstruct/defs`。
*   Python-cim，地址为`https://github.com/fireeye/flare-wmi/blob/master/python-cim/cim/cim.py`和`https://github.com/fireeye/flare-wmi/blob/master/python-cim/cim/objects.py`。
*   Python-sdb，地址为`https://github.com/williballenthin/python-sdb/blob/master/sdb/sdb.py`。~~~~
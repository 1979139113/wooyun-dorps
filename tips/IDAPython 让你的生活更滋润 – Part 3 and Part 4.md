# IDAPython 让你的生活更滋润 – Part 3 and Part 4

0x00 简介
=====

今天在网上看到平底锅blog上Josh Grunzweig发表了一系列关于利用IDAPython分析malware的教程。感觉内容非常不错，于是翻译成中文与大家一起分享。

Part1 和 Part2 的译文地址：  
[http://drops.wooyun.org/papers/11849](http://drops.wooyun.org/papers/11849)

Part3和Par4原文地址：

[http://researchcenter.paloaltonetworks.com/2016/01/using-idapython-to-make-your-life-easier-part-3/](http://researchcenter.paloaltonetworks.com/2016/01/using-idapython-to-make-your-life-easier-part-3/)

[http://researchcenter.paloaltonetworks.com/2016/01/using-idapython-to-make-your-life-easier-part-4/](http://researchcenter.paloaltonetworks.com/2016/01/using-idapython-to-make-your-life-easier-part-4/)

0x01 Part3 - 序
=====

上一篇译文中我们讨论了如何利用Python来方便我们进行逆向，这一篇中我们将会介绍如何利用IDAPython来解决条件断点问题。

当我们用IDA Pro进行调试的时候，我们总想断点在某个满足某些条件的特殊地址上。举两个例子，第一个是我们只想将断点下在某个函数被传递了指定的参数的情况下；第二个是我们想把断点下在某个特定的库被加载到我的虚拟机的时候。今天我将会给大家介绍如何解决这些问题的方法。

0x02 Part3 – 背景介绍
=====

研究员经常会对dll文件进行分析。在很多情况下，这些dll会被其他的可执行文件加载。一种解决方案是用IDA在选项中设置，确保IDA在每个库被加载的时候都暂停下来。如图所示：

![p1](http://drops.javaweb.org/uploads/images/54e5492c09694a13b73c83490e3ec97901c755ca.jpg)

然而这种方式非常低效，经常需要研究人员手动的暂停和继续：先判断最新加载的库是不是想调试的那个，如果不是再继续执行。如果能仅仅运行一条简单的指令，然后坐等想要的文件被加载进来的话，会让我们的调试工作更加惬意。

0x03 Part3 – 条件断点
=====

我们将用一个dd.dll的文件作为例子进行讲解。这个文件在运行过程中被提取后，通常会被利用程序注入到另一个进程中。为了能够在IDA中动态调试这个文件，我将会动态加载rundll32.exe这个保存在system32目录下的执行文件，并且将dd.dll文件作为参数（如图2所示）。在给定导出名的情况下，rundll32.exe文件允许用户加载一个指定的dll文件。在这个样本中，我对dll文件中的Setting函数很有兴趣，于是我将如下参数传递给IDA：

![p2](http://drops.javaweb.org/uploads/images/0ccc80bc8b1a7c48678b1abd6ce79f9652087d16.jpg)

下一步是当一个DLL文件被加载进来后，找到正确的地方设置断点。为了做到这一点，我先在调试器设置中的”在库被加载或卸载时暂停”的选项上打勾，然后我能够看到有一个断点被设置到了NtMapViewOfSection的后面一个指令。

![p3](http://drops.javaweb.org/uploads/images/25adc746c0dcee99bb13fe5b8dbb0768187d3402.jpg)

接下来我在0x7C91ADFB处下了一个断点。在我的代码中，我使用`add_bpt()`和`enable_bpt()`去创建和激活这个断点。

```
'''
ntdll.dll:7C91ADF1 push    0FFFFFFFFh
ntdll.dll:7C91ADF3 push    dword ptr [ebp-20h]
ntdll.dll:7C91ADF6 call    near ptr ntdll_NtMapViewOfSection
ntdll.dll:7C91ADFB mov     edi, eax
'''

address = 0x7C91ADFB # Just after NtMapViewOfSection
add_bpt(address, 0, BPT_SOFT)
enable_bpt(address, True)

```

这个时候，我们已经在0x7C91ADFB处下了断点，并且调试器会在每个dll被加载的时候暂停。为了确保只有当”dd.dll”文件被加载的时候才暂停，我们必须创建一个条件断点。条件断点允许一个研究人员使用IDC或者Python代码去判断一个断点是不是真的被触发。如果返回值为真，那么断点就会被触发，否则将会被忽略。为了使用Python，我们先将Python设置为默认语言。然后，将我们的Python代码保存在一个变量中，然后在SetBptCnd()中调用这个变量，这样就能把我们的代码变成断点的条件判断了。当条件被设置好后，我们让调试器继续运行，直到遇到一个暂停事件。伪代码如下：

```
address = 0x7C91ADFB # Just after NtMapViewOfSection
RunPlugin("python", 3) # Python default programming
StartDebugger("","",""); 

dll = "dd.dll"
condition = """
for m in Modules():
  if "%s".lower() in m.name.lower():
    print "Breaking on", m.name.lower()
    del_bpt(%d)
    return True
return False
""" % (dll, address)

add_bpt(address, 0, BPT_SOFT)
enable_bpt(address, True)
SetBptCnd(address, condition)

continue_process()
GetDebuggerEvent(WFNE_SUSP, -1)

```

实际上用来使用的条件断点的代码如下：

```
for m in Modules():
  if "dd.dll".lower() in m.name.lower():
    print "Breaking on", m.name.lower()
    del_bpt(0x7C91ADFB)
    return True
return False

```

这个代码会遍历所有被加载进来的模块，然后判断”dd.dll”是不是被加载了进来，如果被加载了，一条调试信息将会被打印出来，并且会返回Ture并触发断点。当执行的时候，我们能看到如下输出：

![p4](http://drops.javaweb.org/uploads/images/0fb18dc1dba191e805c11e89b7f4c2001090add8.jpg)

在这时候，我们可以设置我们期望的导出函数为”Setting”然后运行程序直到触发断点。为了能够自动的完成这个任务，如下代码将会被用到：

```
def get_names(base, size, desired_name):
  current_address = base
  while current_address &lt;= base+size:
      current_address = NextHead(current_address)
      print hex(current_address)
      if desired_name in Name(current_address):
        return current_address

for m in Modules():
  if 'dd.dll' in m.name.lower():
    base = m.base
    size = m.size
    analyze_area(base, base+size)
    setting = get_names(base, size, "Setting")
    if setting:
      add_bpt(setting, 0, BPT_SOFT)
      enable_bpt(setting, True)
      continue_process()
GetDebuggerEvent(WFNE_SUSP, -1)

```

这段代码会遍历所有加载的模块来寻找”dd.dll”文件。当找到这个文件之后，我们会分析这个DLL文件并且遍历代码中的函数名。当我们找到”Setting”这个函数名后会在这个函数名对应的函数上设置一个断点。随后继续我们继续运行调试器，直到这个断点被触发。当我们执行的时候，发现调试器暂停在了我们期望的地址上：

![p5](http://drops.javaweb.org/uploads/images/383a07f494cf78e658186f88b85ddc2e401668c3.jpg)

0x04 Part3 – 总结
=====

尽管设置条件断点看起来是个很小的技术，但的确能节省分析人员大量的时间。一小段代码就能解决我们一遍一遍手动的工作，何乐而不为呢。

0x05 Part4 – 序
=====

分析人员经常会遇到复杂度很高的代码，并且在动态运行过程中这些代码并不是显而易见的。使用IDAPython，我们不光可以确定哪些指令被执行了，还可以确定这些指令被执行了多少次。

0x06 Part4 – 背景
=====

为了这篇文章，我写了一个简单的c程序。代码如下：

```
#include "stdafx.h"
#include <stdlib.h>
#include <time.h>

int _tmain(int argc, _TCHAR* argv[])
{
  char* start = "Running the program.";
  printf("[+] %s\n", start);
  char* loop_string = "Looping...";

  srand (time(NULL));
  int bool_value = rand() % 10 + 1;
  printf("[+] %d selected.\n", bool_value);
  if(bool_value > 5)
  {
    char* over_five = "Number over 5 selected. Looping 2 times.";
    printf("[+] %s\n", over_five);
    for(int x = 0; x < 2; x++)
      printf("[+] %s\n", loop_string);
  }
  else
  {
    char* under_five = "Number under 5 selected. Looping 5 times.";
    printf("[+] %s\n", under_five);
    for(int x = 0; x < 5; x++)
      printf("[+] %s\n", loop_string);
  }
  return 0;
}

```

当我们把这个文件载入IDA后，我们可以看到期望的循环和重定向的语句。如果我们不看底层代码（源代码）的话，通过反汇编，我们依然可以知道会发生什么。如图所示：

![p6](http://drops.javaweb.org/uploads/images/63e540ac3e87d4212c91c7058d289ca77096430f.jpg)

但是如果我们想要知道哪一块代码会在运行时执行。我就需要IDAPython来完成这个挑战了。

0x07 Part4 – IDAPython脚本
=====

我们第一个解决的挑战是能够单步处理每一条指令。我们用下面的代码完成这个任务（每一条执行的指令都会通过调试接口输出）：

```
RunTo(BeginEA())
event = GetDebuggerEvent(WFNE_SUSP, -1)

EnableTracing(TRACE_STEP, 1)
event = GetDebuggerEvent(WFNE_ANY|WFNE_CONT, -1)

while True:
  event = GetDebuggerEvent(WFNE_ANY, -1)
  addr = GetEventEa()
  print "Debug: current address", hex(addr), "| debug event", hex(event)
  if event <= 1: break

```

在上面的代码中，我们先启动调试器，然后利用Runto(BeginEA())执行到入口处的代码。随后的GetDebuggerEvent()函数调用将会在断点触发的时候进行等待。然后我们使用EnableTracing()函数启动跟踪。随后的GetDebuggerEvent()将会让调试器继续单步执行。最后我们进入一个循环然后遍历每个地址，直到我们收到进程结束的信号为止。输出结果如下图所示：

![p7](http://drops.javaweb.org/uploads/images/e7d500fe9ca8a8548aa03633ccf0f48c28e31a4c.jpg)

下一步就是取出和设置每一条执行的指令的地址的颜色。为了做到这一点，我们可以使用GetColor()和SetColor()函数。接下来的代码可以得到目标行指令的颜色，然后判断并设置这行指令的颜色。在这个例子中，我使用了四种不同深度的蓝色。某行指令被执行的越多颜色就会越深（我鼓励读者自定义自己的偏好）。

```
def get_new_color(current_color):
  colors = [0xffe699, 0xffcc33, 0xe6ac00, 0xb38600]
  if current_color == 0xFFFFFF:
    return colors[0]
  if current_color in colors:
    pos = colors.index(current_color)
    if pos == len(colors)-1:
      return colors[pos]
    else:
      return colors[pos+1]
  return 0xFFFFFF


current_color = GetColor(addr, CIC_ITEM)
new_color = get_new_color(current_color)
SetColor(addr, CIC_ITEM, new_color)

```

执行上面的代码会让指定行的指令变蓝，如果指定行的指令被执行了多次，会让蓝色变得更深。

![p8](http://drops.javaweb.org/uploads/images/29eebb017a4e28ec7cd2b422bfb9fe01b727bbb6.jpg)

我们可以使用接下的代码去除掉之前在IDA Pro文件中的颜色。因为设置颜色为0xFFFFFF会让颜色变成白色，并且去掉之前设置的所有颜色。

```
heads = Heads(SegStart(ScreenEA()), SegEnd(ScreenEA()))
for i in heads:
  SetColor(i, CIC_ITEM, 0xFFFFFF)

```

将所有的代码放在一起我们可以得到如下结果：

```
heads = Heads(SegStart(ScreenEA()), SegEnd(ScreenEA()))
for i in heads:
    SetColor(i, CIC_ITEM, 0xFFFFFF)

def get_new_color(current_color):
    colors = [0xffe699, 0xffcc33, 0xe6ac00, 0xb38600]
    if current_color == 0xFFFFFF:
        return colors[0]
    if current_color in colors:
        pos = colors.index(current_color)
        if pos == len(colors)-1:
            return colors[pos]
        else:
            return colors[pos+1]
    return 0xFFFFFF

RunTo(BeginEA())
event = GetDebuggerEvent(WFNE_SUSP, -1)

EnableTracing(TRACE_STEP, 1)
event = GetDebuggerEvent(WFNE_ANY|WFNE_CONT, -1)
while True:
    event = GetDebuggerEvent(WFNE_ANY, -1)
    addr = GetEventEa()
    current_color = GetColor(addr, CIC_ITEM)
    new_color = get_new_color(current_color)
    SetColor(addr, CIC_ITEM, new_color)
    if event <= 1: break

```

当我们在我们的程序中执行这段代码的时候，我们能看到如下的变化：我们能看到所有被执行的汇编指令都被高亮了。就像下图所示，指令执行的次数越多，颜色就会越深，这会让我们很容易的理解程序执行的流程。

![p9](http://drops.javaweb.org/uploads/images/135dc4c9a63215ef83ff2c72073e26e7f84477e7.jpg)

0x08 Part4 – 总结
=====

这篇blog介绍了如何利用IDAPython来给程序上色，这个技术能够很好的帮助研究人员分析复杂的代码并节省大量的时间。
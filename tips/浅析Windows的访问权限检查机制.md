# 浅析Windows的访问权限检查机制

**Author: DanielKing**

0x00 简介
=====

在操作系统中，当我们提到安全的时候，意味着有一些资源需要被保护，在Windows操作系统中，这些被保护的资源大多以对象（Object）的形式存在，对象是对资源的一种抽象。每个对象都可以拥有自己的安全描述符（Security Descriptor），用来描述它能够被谁、以何种方式而访问。这些对象是客体，那么访问这些对象的主体是什么呢？这些主体就是操作系统中的各个进程，更准确地说是这些进程中的每个线程。每个进程都有一个基本令牌 (Primary Token)，可以被进程中的每个线程所共享，而有些线程比较特殊，它们正在模拟（Impersonating）某个客户端的身份，因此拥有一个模拟令牌（Impersonation Token），对于这些线程来说，有效令牌就是模拟令牌，而其他线程的有效令牌则是其所属进程的基本令牌。当主体尝试去访问客体时，操作系统会执行访问权限检查（Access Check），具体来说，是用主体的有效令牌与客体的安全描述符进行比对，从而确定该次访问是否合法。为了提高效率，访问权限检查（Access Check）提供了一种缓存机制，即只有当一个对象被一个进程创建或者打开时，才会进行访问权限检查，访问权限检查的结果会被缓存到进程的句柄表（Handle Table）中，该进程对该对象的后续操作只需要查询句柄表中内容即可确定访问的合法性，而不必每次都执行开销更大的访问权限检查（Access Check）。因为对象和进程都有各自的继承层次，所以对象的安全描述符和进程的令牌从逻辑上讲也是可以继承的，比如文件系统中的文件可以继承其所在目录的安全描述符，子进程可以继承父进程的令牌，这种继承机制使让对象和进程拥有了缺省的安全属性，因此，在一个系统中的安全描述符和令牌的实例数目非常有限，大多数情况下是从父辈们继承过来的缺省值。

![模拟线程](http://drops.javaweb.org/uploads/images/128aacdefbbe947247a8c2374d7689be4ba00e89.jpg)

0x01 访问权限检查
=====

访问权限检查包含了多个维度上的检查：

### DACL检查

DACL（自主访问控制表，Discretionary Access Control List）检查是最基本的访问权限检查。

![安全描述符](http://drops.javaweb.org/uploads/images/fdb4033efa1a12d35c9d50d3c97b7dcdccdbdc83.jpg)

在安全描述符中存在着一个DACL表（见安全描述符图解），里面描述了拥有何种身份的主体申请的何种访问权限会被允许或者拒绝。DACL表中的每个表项拥有如下的结构：

```
We'll define the structure of the predefined ACE types.  Pictorally
the structure of the predefined ACE's is as follows:

     3 3 2 2 2 2 2 2 2 2 2 2 1 1 1 1 1 1 1 1 1 1
     1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
    +---------------+-------+-------+---------------+---------------+
    |    AceFlags   | Resd  |Inherit|    AceSize    |     AceType   |
    +---------------+-------+-------+---------------+---------------+
    |                              Mask                             |
    +---------------------------------------------------------------+
    |                                                               |
    +                                                               +
    |                                                               |
    +                              Sid                              +
    |                                                               |
    +                                                               +
    |                                                               |
    +---------------------------------------------------------------+

Mask is the access mask associated with the ACE.  This is either the
access allowed, access denied, audit, or alarm mask.

Sid is the Sid associated with the ACE.

```

其中，对于DACL来说，AceType可以有

```
#define ACCESS_ALLOWED_ACE_TYPE                 (0x0)
#define ACCESS_DENIED_ACE_TYPE                  (0x1)

```

两种可能；Mask是一个代表相关权限集合的位图；主体的身份使用Sid（Security Identifier）来表示，这是一个变长的数据结构：

```
kd> dt nt!_SID
   +0x000 Revision         : UChar
   +0x001 SubAuthorityCount : UChar
   +0x002 IdentifierAuthority : _SID_IDENTIFIER_AUTHORITY
   +0x008 SubAuthority     : [1] Uint4B

```

Sid是Windows安全系统中一个基本类型，可以用来唯一标识多种不同种类的实体，比如标识用户身份与组身份，标识完整性级别、可信赖级别，AppContainer中的Capability等等。

DACL检查过程是这样的，如果对象的安全描述符为空，代表着这个对象不受任何保护，即可以被任何令牌代表的主体访问；如果对象拥有一个非空的安全描述符，但是安全描述符中的Dacl成员为空，代表该对象没有显式地赋予任何主体任何访问权限，即不允许被访问；如果Dacl成员非空，就按照表中的顺序逐一与当前主体的身份Sid进行比对，直到该主体所请求的权限被全部显式允许，或者某一请求的权限被显式地拒绝，检查结束；如果直到Dacl表检查结束，主体申请的权限仍未被全部显式允许，那么仍然代表拒绝。

根据以上规则可知，DACL中各个表项的顺序对访问权限检查的结果至关重要，比如说有一条有关某主体的显式拒绝的表项位于表的最后，如果位于其之前的表项已经显式允许了这一主体申请的权限，那么这条显式拒绝的表项将起不到任何作用。基于此种原因，如果通过Windows右键安全选项菜单添加的DACL表项，显式拒绝的表项永远被置于显式允许表项前面，以保证其有效性。直接通过API添加则不受此种限制。

### 特权（Privileges）以及超级特权（Super Privileges）检查

所谓特权，是指那些不与某一具体对象相关的、影响整个系统的访问权限。而超级特权是这样的一些特权，你一旦拥有其中的一项超级特权，就拥有了取得所有其他特权和权限的潜质，比如SeDebugPrivilege，在一般情况下，一旦拥有该特权，就可以任意读写目标进程的内存，而不仅仅局限于打开该进程获得句柄时得到的那些权限及特权。

### 完整性级别（Integrity Level）和强制策略（Mandatory Policy）检查

完整性级别（IL）机制是Windows Vista引入的重要的安全机制，建立在其基础上的沙盒机制十分简单又异常强大。

### 受限令牌（Restricted Token）检查

通过创建受限令牌，可以获得一个普通令牌所有拥有的权限集合的一个子集，用来进行一些低权限操作，降低安全风险。

### AppContainer's Capabilities检查

LowBox令牌以及AppContainer's Capabilities组身份是从Windows 8开始引入的安全机制，它提供了另一种更加复杂的沙盒机制。Capabilities与Android系统中对于App权限限制非常类似。

### 可信级别（Trust Level）检查

完整性级别检查是一种粗粒度的、相对稳定的访问权限控制手段；伴随着Windows的不断进化，对于签名机制的依赖越来越明显，签名代表了一种可信赖程度。笔者认为，Windows正在逐渐建立一套完善的基于可信赖级别的权限检查机制，受保护进程（Protected Process）就是这是这套机制中一部分。

上面的内容简单介绍了Windows的权限检查模型，以及权限检查的维度；下面将介绍一些实现细节。

0x02 令牌
=====

### TOKEN结构体

Token结构体是访问权限检查中的代表主体身份的核心数据结构，下面展示的是Windows 10 x64平台下的构成。

```
kd> dt nt!_TOKEN
   +0x000 TokenSource      : _TOKEN_SOURCE
   +0x010 TokenId          : _LUID
   +0x018 AuthenticationId : _LUID
   +0x020 ParentTokenId    : _LUID
   +0x028 ExpirationTime   : _LARGE_INTEGER
   +0x030 TokenLock        : Ptr64 _ERESOURCE
   +0x038 ModifiedId       : _LUID
   +0x040 Privileges       : _SEP_TOKEN_PRIVILEGES
   +0x058 AuditPolicy      : _SEP_AUDIT_POLICY
   +0x078 SessionId        : Uint4B
   +0x07c UserAndGroupCount : Uint4B
   +0x080 RestrictedSidCount : Uint4B
   +0x084 VariableLength   : Uint4B
   +0x088 DynamicCharged   : Uint4B
   +0x08c DynamicAvailable : Uint4B
   +0x090 DefaultOwnerIndex : Uint4B
   +0x098 UserAndGroups    : Ptr64 _SID_AND_ATTRIBUTES
   +0x0a0 RestrictedSids   : Ptr64 _SID_AND_ATTRIBUTES
   +0x0a8 PrimaryGroup     : Ptr64 Void
   +0x0b0 DynamicPart      : Ptr64 Uint4B
   +0x0b8 DefaultDacl      : Ptr64 _ACL
   +0x0c0 TokenType        : _TOKEN_TYPE
   +0x0c4 ImpersonationLevel : _SECURITY_IMPERSONATION_LEVEL
   +0x0c8 TokenFlags       : Uint4B
   +0x0cc TokenInUse       : UChar
   +0x0d0 IntegrityLevelIndex : Uint4B
   +0x0d4 MandatoryPolicy  : Uint4B
   +0x0d8 LogonSession     : Ptr64 _SEP_LOGON_SESSION_REFERENCES
   +0x0e0 OriginatingLogonSession : _LUID
   +0x0e8 SidHash          : _SID_AND_ATTRIBUTES_HASH
   +0x1f8 RestrictedSidHash : _SID_AND_ATTRIBUTES_HASH
   +0x308 pSecurityAttributes : Ptr64 _AUTHZBASEP_SECURITY_ATTRIBUTES_INFORMATION
   +0x310 Package          : Ptr64 Void
   +0x318 Capabilities     : Ptr64 _SID_AND_ATTRIBUTES
   +0x320 CapabilityCount  : Uint4B
   +0x328 CapabilitiesHash : _SID_AND_ATTRIBUTES_HASH
   +0x438 LowboxNumberEntry : Ptr64 _SEP_LOWBOX_NUMBER_ENTRY
   +0x440 LowboxHandlesEntry : Ptr64 _SEP_LOWBOX_HANDLES_ENTRY
   +0x448 pClaimAttributes : Ptr64 _AUTHZBASEP_CLAIM_ATTRIBUTES_COLLECTION
   +0x450 TrustLevelSid    : Ptr64 Void
   +0x458 TrustLinkedToken : Ptr64 _TOKEN
   +0x460 IntegrityLevelSidValue : Ptr64 Void
   +0x468 TokenSidValues   : Ptr64 _SEP_SID_VALUES_BLOCK
   +0x470 IndexEntry       : Ptr64 _SEP_LUID_TO_INDEX_MAP_ENTRY
   +0x478 VariablePart     : Uint8B

```

我们比较关注其中的特权位图和三个代表主体身份的Sid数组：UserAndGroups，RestrictedSids，Capabilities。

![_TOKEN结构体](http://drops.javaweb.org/uploads/images/5d1c66fdd6ba7d5c1fe73684e3387a72609cfad5.jpg)

### Privileges位图

特权在用户态是通过LUID来表示，在内核结构体_TOKEN中是使用三个位图来表示:

```
kd> dt _SEP_TOKEN_PRIVILEGES
nt!_SEP_TOKEN_PRIVILEGES
   +0x000 Present          : Uint8B
   +0x008 Enabled          : Uint8B
   +0x010 EnabledByDefault : Uint8B

```

分别代表当前的主体可以选用的特权集合（Present）、已经打开的特权集合（Enabled）和默认打开的特权集合（EnabledByDefault），后两个集合应该是Present集合的子集。

与代表主体身份的Sid数组不同，特权集合的表示简单，而且没有任何保护。从用户态通过API只能打开或者关闭某一项已经存在的特权，而不能增加可选的特权，换句话说，用户态只能修改Enabled特权集合，而不能修改Present特权集合；从内核态可以直接修改Present特权集合，比如给普通进程增加SeAssignPrimaryTokenPrivilege特权，以便为子进程显式指定令牌，而不是继承当前进程的令牌，可以达到扩大子进程权限的效果。

### 代表主体身份的Sid数组

三个代表主体身份的Sid数组分别是：

**UserAndGroups：**

代表着主体的普通用户身份和组身份，是不可或缺的成员；

**RestrictedSids：**

可选成员，如果不为空，则代表着当前的令牌是受限令牌，受限令牌通过从普通令牌中过滤掉一些比较敏感的身份转化而来，受限令牌中的UserAndGroups成员与普通令牌相同，但是RestriectedSids成员不为空，里面保存着过滤后的身份子集；由于在访问权限检查时，三个身份Sid数组要同时进行检查，只有结果都通过才允许该次访问，因此通过增加代表着受限制的权限集合的RestrictedSids成员，既达到了限制令牌权限的目的，又在UserAndGroups成员中保留了原有令牌的完整身份信息。

**Capabilities:**

可选成员，仅用于AppContainer，其中的Sid代表着与App相关的身份，比如拥有连接网络、访问当前位置等权限的身份。

这三个Sid数组都关联了哈希信息，以保护其完整性，因此，即使从内核态直接修改，也会因为无法通过完整性验证而失败。不过好在哈希的算法非常简单，下面展示的就是Windows 10 x64平台下面该算法的C++演示代码：

```
static
NTSTATUS
RtlSidHashInitialize
(
    __in PSID_AND_ATTRIBUTES Groups,
    __in size_t GroupsCount,
    __inout PSID_AND_ATTRIBUTES_HASH HashBuffer
)
{
    if (NULL == HashBuffer)
        return 0xC000000D;

    memset(HashBuffer, 0, 0x110);  
    if (0 == GroupsCount || NULL == Groups)
        return 0;

    HashBuffer->SidCount = GroupsCount;
    HashBuffer->SidAttr = Groups;

    if (GroupsCount > 0x40)
        GroupsCount = 0x40;  
    if (0 == GroupsCount)
        return 0;

    size_t bit_pos = 1;

    for (size_t i = 0; i < GroupsCount; i++)
    {
        PISID sid = reinterpret_cast<PISID>((Groups + i)->Sid); 

        size_t sub_authority_count = sid->SubAuthorityCount;

        DWORD sub_authority = sid->SubAuthority[sub_authority_count - 1];

        *(size_t*)(&HashBuffer->Hash[(sub_authority & 0x0000000F)]) |= bit_pos;
        *(size_t*)(&HashBuffer->Hash[((sub_authority & 0x000000F0) >> 4) + 0x10]) |= bit_pos;

        bit_pos <<= 1;
    }

    return 0;
}

```

该算法有两处明显的局限，只计算每个Sid的最后一个Sub Authority的最低位字节，以及最多只有前64个Sid参与计算。

### UAC与关联令牌

UAC是Vista版本引入的比较重要的安全机制，很多用户抱怨它带来的不便性，然而它就像Linux操作系统中的sudo一样，是保护系统安全的一道重要屏障。当用户登录Windows时，操作系统会为用户生成一对初始令牌，分别是代表着用户所拥有的全部权限的完整版本令牌（即管理员权限令牌），以及被限制管理员权限后的普通令牌，二者互为关联令牌；此后，代表用户的进程所使用的令牌都是由普通令牌继承而来，用来进行常规的、非敏感的操作；当用户需要进行一些需要管理员权限的操作时，比如安装软件、修改重要的系统设置时，都会通过弹出提权对话框的形式提示用户面临的风险，征求用户的同意，一旦用户同意，将会切换到当前普通令牌关联的管理员权限令牌，来进行敏感操作。通过这种与用户交互的方式，避免一些恶意程序在后台稍稍执行敏感操作。

关联令牌是通过Logon Session来实现的，下图展示了其大致原理：

![关联令牌](http://drops.javaweb.org/uploads/images/996aaba83aa2be4d435b09e081c592d893147fde.jpg)

并非所有的令牌都有关联令牌，比如一直运行在较高权限下不需要进行提权操作的令牌。

0x03 对象
=====

### 对象的结构

对象是对操作系统中需要保护和集中管理的资源的一种抽象，对象拥有统一的头部，保存着抽象出来的信息。

```
kd> dt _OBJECT_HEADER
nt!_OBJECT_HEADER
   +0x000 PointerCount     : Int8B
   +0x008 HandleCount      : Int8B
   +0x008 NextToFree       : Ptr64 Void
   +0x010 Lock             : _EX_PUSH_LOCK
   +0x018 TypeIndex        : UChar
   +0x019 TraceFlags       : UChar
   +0x019 DbgRefTrace      : Pos 0, 1 Bit
   +0x019 DbgTracePermanent : Pos 1, 1 Bit
   +0x01a InfoMask         : UChar
   +0x01b Flags            : UChar
   +0x01b NewObject        : Pos 0, 1 Bit
   +0x01b KernelObject     : Pos 1, 1 Bit
   +0x01b KernelOnlyAccess : Pos 2, 1 Bit
   +0x01b ExclusiveObject  : Pos 3, 1 Bit
   +0x01b PermanentObject  : Pos 4, 1 Bit
   +0x01b DefaultSecurityQuota : Pos 5, 1 Bit
   +0x01b SingleHandleEntry : Pos 6, 1 Bit
   +0x01b DeletedInline    : Pos 7, 1 Bit
   +0x01c Spare            : Uint4B
   +0x020 ObjectCreateInfo : Ptr64 _OBJECT_CREATE_INFORMATION
   +0x020 QuotaBlockCharged : Ptr64 Void
   +0x028 SecurityDescriptor : Ptr64 Void
   +0x030 Body             : _QUAD

```

![对象头](http://drops.javaweb.org/uploads/images/2b1262957955491982be264ffcfbcfcee3a05882.jpg)

对象的头部包含着指向安全描述符的指针。

### 安全描述符

安全描述符保存着作为访问权限检查客体的对象的主要信息。它包含着4个关键成员，根据这4个成员是嵌入在结构体的尾部，还是引用自外部缓存位置的差异，可以分为两种不同的结构形式：

```
kd> dt _SECURITY_DESCRIPTOR
ntdll!_SECURITY_DESCRIPTOR
   +0x000 Revision         : UChar
   +0x001 Sbz1             : UChar
   +0x002 Control          : Uint2B
   +0x008 Owner            : Ptr64 Void
   +0x010 Group            : Ptr64 Void
   +0x018 Sacl             : Ptr64 _ACL
   +0x020 Dacl             : Ptr64 _ACL
kd> dt _SECURITY_DESCRIPTOR_RELATIVE
nt!_SECURITY_DESCRIPTOR_RELATIVE
   +0x000 Revision         : UChar
   +0x001 Sbz1             : UChar
   +0x002 Control          : Uint2B
   +0x004 Owner            : Uint4B
   +0x008 Group            : Uint4B
   +0x00c Sacl             : Uint4B
   +0x010 Dacl             : Uint4B

```

Owner和Group代表了该安全描述符的属主Sid，Sacl是系统访问控制表（System Access Control List）的简称，里面可以包含当前对象的审计（Audit）、完整性级别（Integrity Level）以及可信赖级别（Trust Level）等信息，与前面提到的Dacl共用同一结构体。

### 可选头部

对象除了拥有共同的固定头部外，还可以有可选的头部，保存着名称等可选信息。通过固定头部的InfoMask成员查表可以得到可选头部的位置，如图所示：

![可选头部](http://drops.javaweb.org/uploads/images/0b4e92e9ef1596dbaf25610e59defd43b1474636.jpg)

### 对象目录

只有部分对象拥有名称信息，它们被称为命名对象。命名对象的主要作用是方便对象在不同的进程中共享，它们被按类别编纂成对象目录，因此可以通过在对象目录中的路径信息找到该对象。对象目录的实现如下图所示：

![对象目录](http://drops.javaweb.org/uploads/images/ba0cb353ae72f6a635967be1c203a6982590f85c.jpg)

### 对象类型

对象类型是对象中非常重要的信息，Windows将对象的类型信息从同一类对象中抽象出来，保存成一个单独的类型对象。系统中全部的类型对象被集中放置在一个表中，对象通过维护一个指向该表的索引（TypeIndex）来表明当前对象的类型。这个索引值直接保存在对象的头部，而对象体与对象头部直接相邻，如果对象体被损坏，有可能导致头部的索引值被改变，使所谓的类型混淆（Type Confusion）利用成为可能。为了缓解这一问题，Windows 10对该索引值做了特殊保护，如下图所示：

![类型索引保护](http://drops.javaweb.org/uploads/images/00703f53ceb23e1a5d328f3d1c1c0fead322392a.jpg)

这种保护方式简单而强大，新的索引值由3部分经过异或操作得到：类型对象在类型对象表中的真实索引值，对象头部地址的第二个字节（即第8到第15位），保存在ObHeaderCookie全局变量中的因每次系统启动而异的Cookie字节。其中，ObHeaderCookie的引入，使同一类型的对象在不同机器上，甚至是同一机器上两次启动之间的索引值不同，然而这样并不足以缓解类型混淆利用，我们还可以利用信息泄露（Info Leak）来绕过（Bypass）该保护，因此还引入了对象头部地址，使得在同一时刻、同一系统中的两个相同类型对象的不同实例间的索引值也不相同，从而有效地缓解类型混淆利用。

0x04 完整性级别检查
=====

完整性级别（IL）检查是沙盒的最简单实现方式，通过完整性级别在对象和进程层次之间的继承关系，就如同在操作系统中建立起了“世袭制度”。通过严格控制完整性级别的继承规则，以及设置严格的完整性级别检查制度，可以保证“出身低微”的主体无法访问到“出身高贵”的客体资源。

完整性拥有以下几个级别：

```
SeUntrustedMandatorySid
SeLowMandatorySid
SeMediumMandatorySid
SeHighMandatorySid
SeSystemMandatorySid

```

主体的缺省完整性级别是SeUntrustedMandatorySid，而客体的缺省完整性级别是SeMediumMandatorySid，这种差异也进一步地强化了这种“世袭制度”。

0x05 保护进程
=====

保护进程（Protected Process）是Windows操作系统为了保护某些关键进程，防止其被普通进程调试、注入、甚至是读取内存信息而建立起来的一种安全机制。

保护进程与其他普通进程的区别在于，保护进程的Protection成员不为0。

```
kd> dt nt!_EPROCESS Protection
   +0x6b2 Protection : _PS_PROTECTION
kd> dt nt!_PS_PROTECTION
   +0x000 Level            : UChar
   +0x000 Type             : Pos 0, 3 Bits
   +0x000 Audit            : Pos 3, 1 Bit
   +0x000 Signer           : Pos 4, 4 Bits

```

保护进程的Type成员可以代表两种保护类型：Protected Process（PP），Protected Process Lite（PPL），两者的区别在于PP保护进程拥有更高的权限。保护进程的Signer成员代表该进程的签名发布者的类别。对于Signer为PsProtectedSignerWindows（5）和PsProtectedSignerTcb（6）的保护进程，其Type和Signer信息会被抽取出来，组装成一个Sid，代表着该进程的可信赖级别，保存到基本令牌中的TrustLevelSid成员中。当一个令牌中的TrustLevelSid被使用时，需要保证与当前进程的Protection信息保持同步，这主要是为了应对令牌在不同进程间传递时（比如子进程继承父进程的令牌）导致的TrustLevelSid成员过时的情形。

当调试或者创建一个进程时，会调用内核函数PspCheckForInvalidAccessByProtection进行权限检查，该函数根据当前进程以及目标进程的Protection来判定当前操作是否需要遵守保护进程规定的权限限制，具体判定规则如下：

*   如果操作来自于内核态代码，不需要遵守限制；
*   如果目标进程的Protection成员为空，表示目标进程不是保护进程，不需要遵守限制；
*   如果当前进程是PP类型保护进程，该类型保护进程拥有最高权限，不需要遵守限制；
*   如果当前进程与目标进程都是PPL类型保护进程，需要根据RtlProtectedAccess表来判断当前进程的Signer是否优先于（dominate）目标进程的Signer，如果是，不需要遵守限制；
*   其他情况，需要遵守限制。

RtlProtectedAccess表如下所示：

```
RTL_PROTECTED_ACCESS RtlProtectedAccess[] =
{
//   Domination,       Process,         Thread,
//         Mask,  Restrictions,   Restrictions,
    {         0,             0,             0}, //PsProtectedSignerNone               Subject To Restriction Type
    {         2,    0x000fc7fe,    0x000fe3fd}, //PsProtectedSignerAuthenticode       0y00000010
    {         4,    0x000fc7fe,    0x000fe3fd}, //PsProtectedSignerCodeGen            0y00000100
    {         8,    0x000fc7ff,    0x000fe3ff}, //PsProtectedSignerAntimalware        0y00001000
    {      0x10,    0x000fc7ff,    0x000fe3ff}, //PsProtectedSignerLsa                0y00010000
    {      0x3e,    0x000fc7fe,    0x000fe3fd}, //PsProtectedSignerWindows            0y00111110
    {      0x7e,    0x000fc7ff,    0x000fe3ff}, //PsProtectedSignerTcb                0y01111110
};

```

其中，Process Restrictions以及Thread Restrictions分别代表着进程和线程的权限限制，详解如下图：

![保护进程与线程的权限限制](http://drops.javaweb.org/uploads/images/3f97632941f9e0ba136be10b2e77cb6a87a6462b.jpg)

我们可以通过NtDebugActiveProcess以及NtCreateUserProcess的演示代码来理解保护进程及其限制的逻辑：

```
static
NTSTATUS
NtDebugActiveProcess(
    __in HANDLE ProcessHandle,
    __in HANDLE DebugObjectHandle
    )
{
    PEPROCESS target_process = nullptr;
    NTSTATUS result = ObReferenceObjectByHandleWithTag( ProcessHandle, &target_process);
    if (! NT_SUCCESS(result))
        return result;

    PEPROCESS host_process = PsGetCurrentProcess();

    if (host_process == target_process)
        return 0xC0000022;

    if (PsInitialSystemProcess == target_process)
        return 0xC0000022;

    if (PspCheckForInvalidAccessByProtection(PreviousMode, host_process->Protection, target_process->Protection))
        return 0xC0000712;

    // ......
}


static
NTSTATUS
NtCreateUserProcess(
    __out PHANDLE ProcessHandle,
    __out PHANDLE ThreadHandle,
    __in  ACCESS_MASK ProcessDesiredAccess ,
    __in  ACCESS_MASK ThreadDesiredAccess ,
    __in  POBJECT_ATTRIBUTES ProcessObjectAttributes OPTIONAL ,
    __in  POBJECT_ATTRIBUTES ThreadObjectAttributes OPTIONAL ,
    __in  ULONG CreateProcessFlags,
    __in  ULONG CreateThreadFlags,
    __in  PRTL_USER_PROCESS_PARAMETERS ProcessParameters ,
    __in  PVOID Parameter9,
    __in  PNT_PROC_THREAD_ATTRIBUTE_LIST AttributeList
    )
{
    ACCESS_MASK allowed_process_access = ProcessDesiredAccess ;
    ACCESS_MASK allowed_thread_access = ThreadDesiredAccess ;
    PS_PROTECTION protection = ProcessContext.member_at_0x20;
    if (PspCheckForInvalidAccessByProtection(PreviousMode, host_process->Protection, target_process->Protection))
    {
        // 1 << 0x19 = 0x80000,  WRITE_OWNER
        if ( MAXIMUM_ALLOWED == ProcessDesiredAccess )
            allowed_process_access = (((~RtlProtectedAccess[protection.Signer].DeniedProcessAccess) & 0x1FFFFF) | ProcessDesiredAccess) & (~(1 << 0x19));

        if ( MAXIMUM_ALLOWED == ThreadDesiredAccess )
            allowed_thread_access = (((~RtlProtectedAccess[protection.Signer].DeniedThreadAccess) & 0x1FFFFF) | ThreadDesiredAccess) & (~(1 << 0x19));
    }
    //PspInsertProcess(..., allowed_process_access, ...);
    //PspInsertThread(..., allowed_thread_access, ...);
}

```

0x06 结语
=====

Windows操作系统中的权限检查机制以及安全系统一直都在不断地进化着，探索其实现的细节以及推测其背后的设计理念是一件非常有趣有益的事情，本文仅仅是作者在探索中的小结，难免有疏漏甚至错误，更加详细的内容可以参考下列材料，或者直接分析和调试Windows内核，以了解最新最真实的Windows。

*   [James Forshaw: The Windows Sandbox Paradox](http://nullcon.net/website/archives/ppt/goa-15/the-windows-sandbox-paradox.pdf)
*   [James Forshaw: A Link to the Past: Abusing Symbolic Links on Windows](https://vimeo.com/133002251)
*   [Alex Ionescu: The Evolution of Protected Processes Part 1: Pass-the-Hash Mitigations in Windows 8.1](http://www.alex-ionescu.com/?p=97)
*   [Alex Ionescu: The Evolution of Protected Processes Part 2: Exploit/Jailbreak Mitigations, Unkillable Processes and Protected Services](http://www.alex-ionescu.com/?p=116)
*   [Alex Ionescu: Protected Processes Part 3 : Windows PKI Internals (Signing Levels, Scenarios, Root Keys, EKUs & Runtime Signers)](http://www.alex-ionescu.com/?p=146)
*   [Daniel & Azure: Did You Get Your Token?](http://2015.zeronights.ru/assets/files/08-Daniel-Azure.pdf)
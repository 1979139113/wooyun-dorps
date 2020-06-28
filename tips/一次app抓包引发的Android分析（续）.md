# 一次app抓包引发的Android分析（续）

0x00起因
------

* * *

首先说明下，本文不是正统的《一次app抓包引发的Android分析记录》的续篇，只是分析的某APP和起因是一样的，故借题。然而本文所做的分析解决了《一次app抓包引发的Android分析记录》所留下的问题，故称为续。

移动应用已不像起初，burpuite代理改遍天下。交叉编译、传输加密、DEX加壳等防护方式开始再慢慢应用，我们已经很难再肆意的抓包改数据了。

文中所分析的APP也是，将加密算法交叉编译至so库中，对传输参数进行整体加密。分析arm汇编代码是一条路子，但已然令大部分人望而生畏。然而安全总是充满着各种奇思淫意，就算不走汇编，依旧能找到其它路子。

接下来通过这篇文章向各位分享下分析过程。

0x01整体思路
--------

* * *

我们此次分析的目的是为了还原传输过程中的request和response，当然更重要的是要能控制request中的参数。为此我们需要了解参数是如何加密的。通过《一次app抓包引发的Android分析记录》我们知道加密函数是libGoblin.so中的e函数，但似乎通过它要还原出解密函数不是那么容易。

不过既然知道了参数是何函数进行加密，那我们只要再知道参数是什么样的格式，直接调用这个函数来加密伪造后的数据，而后抓包替换掉对应的参数不就可以了吗？

为此我采取的策略是在json数据进入加密函数之前将其截获，这样就能知道参数是以什么样的格式传递。知道了具体的参数格式，便可伪造数据抓包改包。

0x02解析request
-------------

* * *

首先来看下一个request，如下图参数都是加密的，本小节目的就是为了搞清楚下面这一串是什么东西。

![2014091023582979037.png](http://drops.javaweb.org/uploads/images/2bf87c680c52ac69439bbda65a5202dd29723376.jpg)

万年不变的第一步，apktool反编译、dex2jar反编译。当然代码混淆了，不过这不影响我们分析。android发送网络请求主要采用JDK的HttpURLConnection和Apache的HttpClient，搜索跟这两个相关的函数和字段以及请求的URL。而这些特征函数和字段，通常是不会进行混淆的。Andbug动态跟踪亦可，本次通文采用静态代码分析，故如此！

通过AndroidManifest.xml了解app的结构，找到主包名等等。jd-gui查看java代码，request是POST请求，所以搜索post、HttpPost字眼看看：

![2014091023593412283.png](http://drops.javaweb.org/uploads/images/9aa140f59253e942807a7a6f5ecd4ff77a4ee216.jpg)

利用如上方式，反复通过搜索http请求方法及请求URL等特征缩小范围，最终定位到AbstractHttpRequest类，该类是抽象类并拥有很多有趣的方法。而CommonTask类扩展了AbstractHttpRequest类。通过阅读代码知道该类为关键类。《一次app抓包引发的Android分析记录》文中亦说明了，此处不再赘述！

而c参数被AbstractHttpRequest的chrome()处理，跟进该函数：

![2014091100005867568.png](http://drops.javaweb.org/uploads/images/2c87aa2e9cf48ee5d883c55b4595ad9aa9d7c514.jpg)

str为字符串常量“6000lex”和json数据一起进入NetworkParam的convertValue(),跟进：

![2014091100024792775.png](http://drops.javaweb.org/uploads/images/c2e276973c9ea69d96d53a8d986c0fc6ebe2bcdc.jpg)

在convertValue()中最终调用了加密函数，因此我们在Goblin.e加密paramString1和paramString2之前将这两个参数通过log打印出来就知道请求的数据到底是什么东西了。

![2014091100031885901.png](http://drops.javaweb.org/uploads/images/346821bc9910c43e4405c27bc97c8348aac56d61.jpg)

用文本编辑器打开com.xxx.net下的NetworkParam.smali定位到convertValue()函数,在Goblin.e处添加如下代码：

![2014091100041466275.png](http://drops.javaweb.org/uploads/images/a20f40ae55a6e06dffbf8a358cf47682e389c36c.jpg)

通过修改smali代码将加密前后的字符串打印出来，重打包签名后安装，查看logcat日志:

![2014091100045620668.png](http://drops.javaweb.org/uploads/images/57c826e711ecddef270a9e7b5c21d8e62c813c84.jpg)

如上图所示，现在我们已经知道了参数传递的格式了。然而日志中还出现了预料之外的数据。照上分析p1应该为常量“6000lex”，而图中却出现了时间戳。对比传输的数据（下图），发现上图第二个request-e是b参数的密文。而b的明文正是登陆信息。搜索convertValue发现b也是调用该函数加密，故导致此现象。

![2014091100054441062.png](http://drops.javaweb.org/uploads/images/4176110ed5388f56b160b00e6cfb0ea5cca17938.jpg)

通过如上分析，最终确定了登陆时c和b格式分别为：

```
c={“adid":"f854a2765b5dcc3e","cid":"C2487","gid":"5EFD7B7D-A648-2F40-9D53-D42F1BCCC468","ke":"1410276980456","ma":"","mno":"310260","model":"sdk","msg":"","nt":"burp","osVersion":"4.4_19","pid":"10010","sid":"352C4C51-F09F-C929-E03A-B5E311BA2808","t":"p_ucLogin","uid":"000000000000000","un":"","vid":"60001060"} 

b={"loginT":1,"paramJson":"","prenum":"","pwd":"xxxx","uname":"xxxx","userSecurity":{"communityCode":"0","imeiCode":"000000000000000","imsiCode":"310260000000000","osType":"14","stationId":"0","terminalType":"02"}}

```

由于c中的ke和b的加密因子一样。猜测服务端解密时，先通过“6000lex”解密出c，获取ke的值，再通过ke解密出b参数。

实际上上面那个猜想是正确的，而《一次app抓包引发的Android分析记录》中所做的猜想也是正确的。Goblin.d()就是解密函数。

《一次app抓包引发的Android分析记录》中所记录的：

![2014091100063691127.png](http://drops.javaweb.org/uploads/images/63d129848da9a97e87aa50574f0d54c0486a737b.jpg)

利用Goblin.d()解密后为：

![2014091100070015786.png](http://drops.javaweb.org/uploads/images/9b5b79ee44570fc6918c79d06da6a5a353adbe39.jpg)

至于《一次app抓包引发的Android分析记录》中为何解密会失败，下面章节会说明到！

0x03解析response
--------------

* * *

如下，response响应报文亦是乱七八糟的一堆。这样就算我们能改包了，也无法判断结果到底是否正确。因此在开始改包之前，必须先解密response。

![2014091100090539984.png](http://drops.javaweb.org/uploads/images/b7ccb57ba05d685689a827046eea61872def165f.jpg)

通过0x02的分析知道app是用Apache的HttpClient进行post请求。而HttpClient获取response报文是通过getEntity()函数。故直接搜索getEntity，有了0x02的分析，轻松定位到AbstractHttpRequest类的getResult方法，由于此处dex2jar反编译错误。所以直接分析smali代码。

打开AbstractHttpRequest.smali:

![2014091100094127866.png](http://drops.javaweb.org/uploads/images/29802284509e98e5cb939ee02a0d995921126506.jpg)

![2014091100101178907.png](http://drops.javaweb.org/uploads/images/7b7e98fea1e66464bc7aec36d502c023db4e0aa8.jpg)

getEntity()结果为v0，v0最终进入到dealWithResponse(),而dealWithResponse返回一个Object而不是String，所以继续跟进dealWithResponse():

![2014091100104472979.png](http://drops.javaweb.org/uploads/images/d690340c75b876950bf9be018d3f08a44f9cb141.jpg)

dealWithResponse只有一个参数，故跟踪p1:

![2014091100111966691.png](http://drops.javaweb.org/uploads/images/824d0a23c5ab8e1c2ca478e5530a9a2df42e86c4.jpg)

p1最终进行到parseProtoResponse()和buildHttpResultString()中,跟进parseProtoResponse()：

![2014091100120155024.png](http://drops.javaweb.org/uploads/images/13f0b6f16915500b316fae5ad6a35522c904f94f.jpg)

parseProtoResponse()中调用了Goblin中的函数，可能这个函数就是我们想要的。先放着。我们再看看buildHttpResultString()这个函数，此处直接jd-gui查看：

![2014091100123887602.png](http://drops.javaweb.org/uploads/images/a1029a29be8195ce5affa85f775b640767061d67.jpg)

buildHttpResultString()是个抽象类，具体代码在CommonTask、MultiTask、PollTask中实现。而这几个类中最终都是调用Response类的pareResponse()方法，而pareResponse()也调用了Goblin中的解密函数。parseProtoResponse()也是Response类一个方法。所以我们将解密响应报文的函数定位在这两个函数中。接下来就是验证下是否正确。

修改Response.smali中的parseProtoResponse()和pareResponse()，将解密后的结果通过日志打印出来：

![2014091100131736750.png](http://drops.javaweb.org/uploads/images/c18cf365c51fe1eb0cae07bbe9880e84c886bcf0.jpg)

在parseProtoResponse()和pareResponse()中标记了几处，但最终只出现如上结果，故确认解密函数为pareResponse()。

0x04控制request
-------------

* * *

至此我们已经完全看到了request和response传输的明文内容，如下所示，一个完成的请求响应过程：

![2014091100141294546.png](http://drops.javaweb.org/uploads/images/9a9f74b7e9638e9d8e4be0ec8c8b3357e20fa3fc.jpg)

当然我们目的是为了测试，所以必须要能改request请求才可以。

尝试一：

既然我们知道了request的加密函数，那么在自己的APP中调用，加密完替换掉burp拦截到得数据即可。

新建一个app引用同样的Goblin类，进行加解密测试，测试代码如下：

![2014091100145140718.png](http://drops.javaweb.org/uploads/images/10e72dcf640c741ec6c92897660cfe968f1c1f4c.jpg)

加密可以加密，但实际上加密结果和原始的密文不一样。而解密却失败，失败的原因正和《一次app抓包引发的Android分析记录》中所做的测试一样。

![2014091100151565721.png](http://drops.javaweb.org/uploads/images/128f3803465dc572cd1652aa17a11bf91a7dfafe.jpg)

发生了JNI WARNING：NewStringUTF input is not valid Modified UTF-8错误，而这个错误是由于在JNI中，google修改了UTF8的标准，当正常UTF8中包含了不符合这个标准的字节时，checkJNI函数就会报这个错，导致应用崩溃。

在google中亦有关于此错误的报告：

https://code.google.com/p/android/issues/detail?id=64892        

https://code.google.com/p/android/issues/detail?id=25386

然而直接将原始密文拿来解密却可以正常解码，所以问题出可能出在加密环节。折腾半天，没有发现适合我们这边的处理办法。但在测试过程中发现，在原始APP应用中通过修改smali代码可以正常加解密。难道so中还有检测环境？当然这个得查看arm汇编才知道。

既然如此，那直接改造原始app吧。

尝试二：

改造思路：在原始APP内部中拦截参数—>通过外部修改拦截到的参数—>放行参数，后续按正常流程进行

在原始app内部中拦截参数，只要在让参数再进入加密之前进入到我们控制的函数中即可。为了方便说明。我们先来看看如何通过外部修改内部拦截的参数。

借助android的广播机制，我们能实现实时的跟app进行交互。因此，也能实时的让外部跟app内部进行数据交互。新建一个myBroadcast类，代码如下：

```
public class myBroadcast extends BroadcastReceiver{ public static boolean sw = false; public static boolean swc = false; public static boolean swb = false; public static String datac = null; public static String datab = null; public static String kec = "6000lex";

    // 接收广播
    public void onReceive(Context context, Intent intent) {     
        Log.i("broadcast-intent",intent.toString());

        String action = intent.getAction();
        if(action.equals("com.test.broadcast1")){
            sw = intent.getBooleanExtra("sw", true);   // 控制是否拦截
            swc = intent.getBooleanExtra("swc", false);// 控制拦截c      
            swb = intent.getBooleanExtra("swb",false); // 控制拦截b

        }
        else if(action.equals("com.test.broadcast2")){
            datac = intent.getStringExtra("datac");    // 接收c
            datab = intent.getStringExtra("datab");    // 接收b
        }       
        Log.v("receiver-data:sw:swc:swb|datac:datab",sw+":"+swc+":"+swb+"|"+datac+":"+datab);
    }

    // 延迟函数
    public static void delay(){     
        try {
            Thread.currentThread();
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    // 修改函数
    public static String alter(String ke,String json){
        Log.v("json-data-1",ke+":"+json);

        while(sw){
            delay();
            if(ke.equals(kec)){
                if(swc){
                    if(datac!=null){
                        json = datac;
                        Log.v("json-data1-2",json);
                        break;
                    }                                           
                }
                else{
                    break;
                }
            }
            else{
                if(swb){
                    if(datab!=null){
                        json =datab;
                        Log.v("json-data2-2",json);
                        break;                  
                    }
                }
                else{
                    break;
                }
            }                       
        }   
        return json;
    }

}

```

利用onReceive实时接收外部数据，利用alter函数检测外部是否发送了拦截指令。当收到拦截的指令时，调用delay()函数进行循环延时，直到接收到伪造的数据或者关闭拦截。

将myBroadcast类转化为smali代码。把myBroadcast.smali放在NetworkParam.smali同级目录下。并在AndroidManifest.xml中注册myBroadcast这个receiver，且将exproted设置为true。

即允许该receiver被外部访问。

![2014091100175256805.png](http://drops.javaweb.org/uploads/images/4459b70657d04ad6fdd59192b12f5330bac135aa.jpg)

这样我们就能在NetworkParam.smali中使用这个receiver

接下来，我们来修改NetworkParam.smali，使其接收我们的指令。在Goblin.e前后修改如下：

![2014091100181216605.png](http://drops.javaweb.org/uploads/images/1b4f3e65b06457bb96eaf02102fef049acaa5a4a.jpg)

让p0进入alter()函数，修改后结果保存至v3，让v3替换p0进入到Goblin.e()中。“request-data-1”打印出原来的参数，“json-data-1”打印出进入alter函数中的参数。“request-data-2”打印出修改后的参数，“request-data-e”，打印出加密后的参数。

修改完后，重新打包、签名、安装，运行查看logcat日志：

1、 默认设置是不拦截，所以logcat日志中应该能完整得看到

```
“request-data-1”—>“json-data-1”—>“request-data-2(不变)”—>“request-data-e”—>“response-data”

```

测试如下：

![2014091100184047629.png](http://drops.javaweb.org/uploads/images/78365267089fd32200a0542442ab3b7e52b72e26.jpg)

和预期一样

2、 设置拦截指令，即sw为true，拦截数据进入循环延迟并等待接收伪造的参数。所以logcat日志中应该只能看到“request-data-1”—>”json-data-1”,测试如下：

am命令发送广播：

![2014091100191488877.png](http://drops.javaweb.org/uploads/images/a1150728a6091431793f89153d0e3af6d9d02268.jpg)

Logcat日志，如预期，数据被拦截，应用一直处于加载当中：

![2014091100193347858.png](http://drops.javaweb.org/uploads/images/e87bc6ed81edfa803e3d15e7d4224269b7487d87.jpg)

3、发送伪造数据，按照设计。此时logcat日志中应该能看到

```
“request-data-1”—>“json-data-1”—>“request-data-2(修改后)”—>“request-data-e”—>“response-data”

```

测试如下：

发送广播指令，拦截b参数,放行c参数：

![2014091100195565253.png](http://drops.javaweb.org/uploads/images/b9874dd1d07f93073f8700d3ecbdfa501471f435.jpg)

Logcat日志显示如下，c(带600lex的)被放行，b(带时间戳的)被拦截：

![2014091100202519355.png](http://drops.javaweb.org/uploads/images/58359a90b85a675f216a785ff26cfff6cec0a620.jpg)

am命令发送datab数据，将uname改为test222：

![2014091100220530381.png](http://drops.javaweb.org/uploads/images/76f1c6ebd0cbd8f4f74da555fca2895581d3bfa2.jpg)

结果如下，receiver接收到伪造的数据后，将原b参数进行修改后放行：

![2014091100223140482.png](http://drops.javaweb.org/uploads/images/21299dcf311095e17b72e7dbc8e9552d51a2c7af.jpg)

和预期一样。

0x05结语
------

* * *

最终，我们实现了查看传输明文信息，并控制了request请求报文。即便它采用so加密。我们依旧可以尽情的测试了~
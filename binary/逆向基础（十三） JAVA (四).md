# 逆向基础（十三） JAVA (四)

54.15 异常 让我们稍微修改一下，月处理的那个例子(在932页的54.13.4)
=====

清单 54.10: IncorrectMonthException.java

```
public class IncorrectMonthException extends Exception
{
   private int index;
   public IncorrectMonthException(int index)
   {
      this.index = index;
   }
   public int getIndex()
   {
      return index;
   }
}

```

清单 54.11: Month2.java

```
class Month2
{
    public static String[] months =
    {
    "January",
    "February",
    "March",
    "April",
    "May",
    "June",
    "July",
    "August",
    "September",
    "October",
    "November",
    "December"
    };

    public static String get_month (int i) throws ⤦
    Ç IncorrectMonthException
    {
            if (i<0 || i>11)
                throw new IncorrectMonthException(i);
                return months[i];
    };
    public static void main (String[] args)
    {
        try
        {
            System.out.println(get_month(100));
        }
        catch(IncorrectMonthException e)
        {
            System.out.println("incorrect month ⤦
            Ç index: "+ e.getIndex());
            e.printStackTrace();
        }
    };
}

```

本质上，IncorrectMonthExceptinClass类只是做了对象构造，还有访问器方法。 IncorrectMonthExceptinClass是继承于Exception类，所以，IncorrectMonth类构造之前，构造父类Exception，然后传递整数给IncorrectMonthException类作为唯一的属性值。

```
public IncorrectMonthException(int);
flags: ACC_PUBLIC
Code:
stack=2, locals=2, args_size=2
0: aload_0
1: invokespecial #1 // Method java/⤦
Ç lang/Exception."<init>":()V
4: aload_0
5: iload_1
6: putfield #2 // Field index:I
9: return

```

getIndex()只是一个访问器，引用到IncorrectMothnException类，被传到LVA的0槽(this指针),用aload_0指令取得， 用getfield指令取得对象的整数值，用ireturn指令将其返回。

```
public int getIndex();
flags: ACC_PUBLIC
Code:
stack=1, locals=1, args_size=1
0: aload_0
1: getfield #2 // Field index:I
4: ireturn

```

现在来看下month.class的get_month方法。

清单 54.12: Month2.class

```
public static java.lang.String get_month(int) throws ⤦
Ç IncorrectMonthException;
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=1, args_size=1
0: iload_0
1: iflt 10
4: iload_0
5: bipush 11
7: if_icmple 19
10: new #2 // class ⤦
Ç IncorrectMonthException
13: dup
14: iload_0
15: invokespecial #3 // Method ⤦
Ç IncorrectMonthException."<init>":(I)V
18: athrow
19: getstatic #4 // Field months:[⤦
Ç Ljava/lang/String;
22: iload_0
23: aaload
24: areturn
949

```

iflt 在行偏移1 ，如果小于的话，

这种情况其实是无效的索引，在行偏移10创建了一个对象，对象类型是作为操作书传递指令的。（这个IncorrectMonthException的构造届时，下标整数是被通过TOS传递的。行15偏移） 时间流程走到了行18偏移，对象已经被构造了，现在athrow指令取得新构对象的引用，然后发信号给JVM去找个合适的异常句柄。

athrow指令在这个不返回到控制流，行19偏移的其他的个基本模块，和异常无关，我们能得到到行7偏移。 句柄怎么工作？ main()在inmonth2.class

清单 54.13: Month2.class

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=2, args_size=1
0: getstatic #5 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
3: bipush 100
5: invokestatic #6 // Method ⤦
Ç get_month:(I)Ljava/lang/String;
8: invokevirtual #7 // Method java/io⤦
Ç /PrintStream.println:(Ljava/lang/String;)V
11: goto 47
14: astore_1
15: getstatic #5 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
18: new #8 // class java/⤦
Ç lang/StringBuilder
21: dup
22: invokespecial #9 // Method java/⤦
Ç lang/StringBuilder."<init>":()V
25: ldc #10 // String ⤦
Ç incorrect month index:
27: invokevirtual #11 // Method java/⤦
Ç lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/⤦
Ç StringBuilder;
30: aload_1
31: invokevirtual #12 // Method ⤦
Ç IncorrectMonthException.getIndex:()I
34: invokevirtual #13 // Method java/⤦
Ç lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
37: invokevirtual #14 // Method java/⤦
Ç lang/StringBuilder.toString:()Ljava/lang/String;
40: invokevirtual #7 // Method java/io⤦
Ç /PrintStream.println:(Ljava/lang/String;)V
43: aload_1
44: invokevirtual #15 // Method ⤦
Ç IncorrectMonthException.printStackTrace:()V
47: return
Exception table:
from to target type
0 11 14 Class IncorrectMonthException

```

950 这是一个异常表，在行偏移0-11（包括）行，一个IncorrectinMonthException异常可能发生，如果发生，控制流到达14行偏移，确实main程序在11行偏移结束，在14行异常开始， 没有进入此区域条件(condition/uncondition)设定，是不可能到打这个位置的。（PS：就是没有异常捕获的设定，就不会有异常流被调用执行。）

但是JVM会传递并覆盖执行这个异常case。 第一个astore_1(在行偏移14)取得，将到来的异常对象的引用，存储在LVA的槽参数1之后。getIndex()方法（这个异常对象） 会被在31行偏移调用。引用当前的异常对象，是在30行偏移之前。 所有的这些代码重置都是字符串操作代码：第一个整数值使用的是getIndex()方法，被转换成字符串使用的是toString()方法，它会和“正确月份下标”的文本字符来链接（像我们之前考虑的那样）。 println()和printStackTrace(1)会被调用，PrintStackTrace(1)调用 结束之后，异常被捕获，我们可以处理正常的函数，在47行偏移，return结束main（）函数 , 如果没有发生异常，不会执行任何的代码。

这有个例子，IDA是如何显示异常范围：

清单54.14 我从我的计算机中找到 random.class 这个文件

```
.catch java/io/FileNotFoundException from met001_335 to ⤦
Ç met001_360\
using met001_360
.catch java/io/FileNotFoundException from met001_185 to ⤦
Ç met001_214\
using met001_214
.catch java/io/FileNotFoundException from met001_181 to ⤦
Ç met001_192\
using met001_195
951
CHAPTER 54. JAVA 54.16. CLASSES
.catch java/io/FileNotFoundException from met001_155 to ⤦
Ç met001_176\
using met001_176
.catch java/io/FileNotFoundException from met001_83 to ⤦
Ç met001_129 using \
met001_129
.catch java/io/FileNotFoundException from met001_42 to ⤦
Ç met001_66 using \
met001_69
.catch java/io/FileNotFoundException from met001_begin to ⤦
Ç met001_37\
using met001_37

```

54.16 类 简单类
=====

清单 54.15: test.java

```
public class test
{
    public static int a;
    private static int b;
    public test()
    {
        a=0;
        b=0;
    }
    public static void set_a (int input)
    {
        a=input;
    }
    public static int get_a ()
    {
        return a;
    }
    public static void set_b (int input)
    {
        b=input;
    }
    public static int get_b ()
    {
        return b;
    }
}

```

构造函数，只是把两个之段设置成0.

```
public test();
flags: ACC_PUBLIC
Code:
stack=1, locals=1, args_size=1
0: aload_0
1: invokespecial #1 // Method java/⤦
Ç lang/Object."<init>":()V
4: iconst_0
5: putstatic #2 // Field a:I
8: iconst_0
9: putstatic #3 // Field b:I
12: return

```

a的设定器

```
public static void set_a(int);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=1, locals=1, args_size=1
0: iload_0
1: putstatic #2 // Field a:I
4: return

```

a的取得器

```
public static int get_a();
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=1, locals=0, args_size=0
0: getstatic #2 // Field a:I
3: ireturn

```

b的设定器

```
public static void set_b(int);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=1, locals=1, args_size=1
0: iload_0
1: putstatic #3 // Field b:I
4: return

```

b的取得器

```
public static int get_b();
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=1, locals=0, args_size=0
0: getstatic #3 // Field b:I
3: ireturn

```

953 类中的公有和私有字段代码没什么区别。 但是类型信息会在in.class 文件中表示，并且，无论如何私有变量是不可以被访问的。

让我们创建对象并调用方法： 清单 54.16: ex1.java

954 新指令创建对象，但不调用构造函数（它在4行偏移被调用）set_a()方法被在16行偏移被调用，字段访问使用的getstatic指令,在行偏移21。

Listing 54.16: ex1.java

```
public class ex1
{
    public static void main(String[] args)
    {
        test obj=new test();
        obj.set_a (1234);
        System.out.println(obj.a);
    }
}

```

* * *

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=2, locals=2, args_size=1
0: new #2 // class test
3: dup
4: invokespecial #3 // Method test."<⤦
Ç init>":()V
7: astore_1
8: aload_1
9: pop
10: sipush 1234
13: invokestatic #4 // Method test.⤦
Ç set_a:(I)V
16: getstatic #5 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
19: aload_1
20: pop
21: getstatic #6 // Field test.a:I
24: invokevirtual #7 // Method java/io⤦
Ç /PrintStream.println:(I)V
27: return

```

54.17 简单的补丁。
------------

* * *

54.17.1 第一个例子

让我们进入简单的一个例子。

```
public class nag
{
    public static void nag_screen()
    {
        System.out.println("This program is not ⤦
            Ç registered");
    };
    public static void main(String[] args)
    {
        System.out.println("Greetings from the mega-⤦
            Ç software");
        nag_screen();
    }
}

```

我们如何去除打印输出"This program is registered".

最会在IDA中加载.class文件。

清单54.1: IDA

![enter image description here](http://drops.javaweb.org/uploads/images/704563bbb0f2f23d5a05d7773a723e319e8c8fa2.jpg)

我们patch到函数的第一个自己到177(返回指令操作码) Figure 54.2 : IDA

![enter image description here](http://drops.javaweb.org/uploads/images/9d9297be929d246088ed1505ddd85bbc8624163f.jpg)

这个在JDK1.7中不工作

```
Exception in thread "main" java.lang.VerifyError: Expecting a ⤦
Ç stack map frame
Exception Details:
Location:
nag.nag_screen()V @1: nop
Reason:
Error exists in the bytecode
Bytecode:
0000000: b100 0212 03b6 0004 b1
at java.lang.Class.getDeclaredMethods0(Native Method)
at java.lang.Class.privateGetDeclaredMethods(Class.java⤦
Ç :2615)
at java.lang.Class.getMethod0(Class.java:2856)
at java.lang.Class.getMethod(Class.java:1668)
at sun.launcher.LauncherHelper.getMainMethod(⤦
Ç LauncherHelper.java:494)
at sun.launcher.LauncherHelper.checkAndLoadMain(⤦
Ç LauncherHelper.java:486)

```

956 也许，JVM有一些检查，关联到栈映射。 好的，为了让path不同，我们使用remove call nag()

清单:54.5 IDA NOP的操作码是0: 现在工作起来了

![enter image description here](http://drops.javaweb.org/uploads/images/d975a81aee9701c0a6b4d50e42d682957985b85d.jpg)

54.17.2第二个例子

现在是另外一个简单的crackme例子。

```
public class password
{
    public static void main(String[] args)
    {
        System.out.println("Please enter the password")⤦
            Ç ;
    String input = System.console().readLine();
    if (input.equals("secret"))
        System.out.println("password is correct⤦");
    else
        System.out.println("password is not ⤦
        Ç correct");
    }
}

```

图54.4:IDA

![enter image description here](http://drops.javaweb.org/uploads/images/58e62b4b30cd942293ed5c3a34ed84b1e7f4d94f.jpg)

我们看ifeq指令是怎么工作的，他的名字的意思是如果等于。 这是不恰当的，我更愿意命名if (ifz if zero) 如果栈顶值是0，他就会跳转，在我们这个例子，如果密码 不正确他就跳转。（equal方法返回的是0） 首先第一个主意是patch这个指令... iefq是两个bytes的操作码 编码和跳转偏移，让这个指令定制，我们必须设定byte3 3byte（因为3是添加到当前地址结束总是跳转同下一条指令） 因为ifeq的指令长度是3bytes.

958 图54.5IDA

![enter image description here](http://drops.javaweb.org/uploads/images/a96bbd61384896af499e85ba65ec859e490be659.jpg)

这个在JDK1.7中不工作

```
Exception in thread "main" java.lang.VerifyError: Expecting a ⤦
Ç stackmap frame at branch target 24
Exception Details:
Location:
password.main([Ljava/lang/String;)V @21: ifeq
Reason:
Expected stackmap frame at this location.
Bytecode:
0000000: b200 0212 03b6 0004 b800 05b6 0006 4c2b
0000010: 1207 b600 0899 0003 b200 0212 09b6 0004
0000020: a700 0bb2 0002 120a b600 04b1
Stackmap Table:
append_frame(@35,Object[#20])
same_frame(@43)
at java.lang.Class.getDeclaredMethods0(Native Method)
at java.lang.Class.privateGetDeclaredMethods(Class.java⤦
Ç :2615)
at java.lang.Class.getMethod0(Class.java:2856)
at java.lang.Class.getMethod(Class.java:1668)
at sun.launcher.LauncherHelper.getMainMethod(⤦
Ç LauncherHelper.java:494)
959
CHAPTER 54. JAVA 54.18. SUMMARY
at sun.launcher.LauncherHelper.checkAndLoadMain(⤦
Ç LauncherHelper.java:486)

```

不用说了，它工作在JRE1.6 我也尝试替换所有3ifeq操作吗bytes，使用0字节(NOP)，它仍然会工作，好 可能没有更多的堆栈映射在JRE1.7中被检查出来。

好的，我替换整个equal调用方法，使用icore_1指令加上NOPS的争强patch.

![enter image description here](http://drops.javaweb.org/uploads/images/f897d164d23c32bd329b10716ac32c328e4d677b.jpg)

11总是在栈顶，当ifeq指令别执行...所以ifeq不会被执行。

工作了。

54.18总结

960 和C/C+比较java少了一些什么？ 结构体：使用类 联合：使用集团类 无附加数据类型，多说一句，还有一些在Java中实现的加密算法的硬编码。 函数指针。

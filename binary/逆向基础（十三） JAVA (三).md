# 逆向基础（十三） JAVA (三)

54.13数组
=====

54.13.1简单的例子 我们首先创建一个长度是10的整型的数组，对其初始化。

```
public static void main(String[] args)
{
    int a[]=new int[10];
    for (int i=0; i<10; i++)
    a[i]=i;
    dump (a);
}

```

* * *

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=3, args_size=1
0: bipush 10
2: newarray int
4: astore_1
5: iconst_0
6: istore_2
7: iload_2
8: bipush 10
10: if_icmpge 23
13: aload_1
14: iload_2
15: iload_2
16: iastore
17: iinc 2, 1
20: goto 7
23: aload_1
24: invokestatic #4 // Method dump:([⤦
Ç I)V
27: return

```

newarray指令，创建了一个有10个整数元素的数组，数组的大小设置使用bipush指令，然后结果会返回到栈顶。数组类型用newarry指令操作符，进行设定。

newarray被执行后，引用（指针）到新创建的数据，栈顶的槽中，astore_1存储引用指向到LVA的一号槽，main()函数的第二个部分，是循环的存储值1到相应的素组元素。 aload_1得到数据的引用并放入到栈中。lastore将integer值从堆中存储到素组中，引用当前的栈顶。main()函数代用dump()的函数部分，参数是，准备给aload_1指令的（行偏移23）

现在我们进入dump()函数。

```
public static void dump(int a[])
{
    for (int i=0; i<a.length; i++)
    System.out.println(a[i]);
}

```

* * *

```
public static void dump(int[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=2, args_size=1
0: iconst_0
1: istore_1
2: iload_1
3: aload_0
4: arraylength
5: if_icmpge 23
8: getstatic #2 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
11: aload_0
12: iload_1
13: iaload
14: invokevirtual #3 // Method java/io⤦
Ç /PrintStream.println:(I)V
17: iinc 1, 1
20: goto 2
23: return

```

到了引用的数组在0槽，a.length表达式在源代码中是转化到arraylength指令，它取得数组的引用，并且数组的大小在栈顶。 iaload在行偏移13被用于装载数据元素。 它需要在堆栈中的数组引用。用aload_0 11并且索引（用iload_1在行偏移12准备）

无可厚非，指令前缀可能会被错误的理解，就像数组指令，那样不正确，这些指令和对象的引用一起工作的。数组和字符串都是对象。

54.13.2 数组元素的求和

另外的例子

```
public class ArraySum
{
    public static int f (int[] a)
    {
        int sum=0;
        for (int i=0; i<a.length; i++)
        sum=sum+a[i];
        return sum;
    }
}

```

* * *

```
public static int f(int[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=3, args_size=1
0: iconst_0
1: istore_1
2: iconst_0
3: istore_2
4: iload_2
5: aload_0
6: arraylength
7: if_icmpge 22
10: iload_1
11: aload_0
12: iload_2
13: iaload
14: iadd
15: istore_1
16: iinc 2, 1
19: goto 4
22: iload_1
23: ireturn

```

LVA槽0是数组的引用，LVA槽1是本地变量和。

54.13.3 main（）函数唯一的数据参数

让我们使用唯一的main()函数参数，字符串数组。

```
public class UseArgument
{
    public static void main(String[] args)
    {
        System.out.print("Hi, ");
        System.out.print(args[1]);
        System.out.println(". How are you?");
    }
}

```

934 0参（argument）第0个参数是程序（和C/C++类似）

因此第一个参数，而第一参数是拥护提供的。

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=1, args_size=1
0: getstatic #2 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
3: ldc #3 // String Hi,
5: invokevirtual #4 // Method java/io⤦
Ç /PrintStream.print:(Ljava/lang/String;)V
8: getstatic #2 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
11: aload_0
12: iconst_1
13: aaload
14: invokevirtual #4 // Method java/io⤦
Ç /PrintStream.print:(Ljava/lang/String;)V
17: getstatic #2 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
20: ldc #5 // String . How ⤦
Ç are you?
22: invokevirtual #6 // Method java/io⤦
Ç /PrintStream.println:(Ljava/lang/String;)V
25: return

```

aload_0在11行加载，第0个LVA槽的引用（main（）函数唯一的参数） iconst_1和aload在行偏移12,13，取得数组第一个元素的引用（从0计数） 字符串对象的引用在栈顶行14行偏移，给println方法。

54.1.34 初始化字符串数组

```
class Month
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
    public String get_month (int i)
    {
        return months[i];
    };
}

```

get_month()函数很简单

```
public java.lang.String get_month(int);
flags: ACC_PUBLIC
Code:
stack=2, locals=2, args_size=2
0: getstatic #2 // Field months:[⤦
Ç Ljava/lang/String;
3: iload_1
4: aaload
5: areturn

```

aaload操作数组引用，java字符串是一个对象，所以a_instructiong被用于操作他们.areturn返回字符串对象的引用。

month[]数值是如果初始化的？

```
static {};
flags: ACC_STATIC
Code:
stack=4, locals=0, args_size=0
0: bipush 12
2: anewarray #3 // class java/⤦
Ç lang/String
5: dup
6: iconst_0
7: ldc #4 // String January
9: aastore
10: dup
11: iconst_1
12: ldc #5 // String ⤦
Ç February
14: aastore
15: dup
16: iconst_2
17: ldc #6 // String March
19: aastore
20: dup
21: iconst_3
22: ldc #7 // String April
24: aastore
25: dup
26: iconst_4
27: ldc #8 // String May
29: aastore
30: dup
31: iconst_5
32: ldc #9 // String June
34: aastore
35: dup
36: bipush 6
38: ldc #10 // String July
40: aastore
41: dup
42: bipush 7
44: ldc #11 // String August
46: aastore
47: dup
48: bipush 8
50: ldc #12 // String ⤦
Ç September
52: aastore
53: dup
54: bipush 9
56: ldc #13 // String October
58: aastore
59: dup
60: bipush 10
62: ldc #14 // String ⤦
Ç November
64: aastore
65: dup
66: bipush 11
68: ldc #15 // String ⤦
Ç December
70: aastore
71: putstatic #2 // Field months:[⤦
Ç Ljava/lang/String;
74: return

```

937 anewarray 创建一个新数组的引用（a是一个前缀）对象的类型被定义在anewarray操作数中，它在这是“java/lang/string”文本字符串，在这之前的bipush 1L是设置数组的大小。 对于我们再这看到一个新指令dup，他是一个众所周知的堆栈操作的计算机指令。用于复制栈顶的值。（包括了之后的编程语言）它在这是用于复制数组的引用。因为aastore张玲玲 起到弹出堆栈中的数组的作用，但是之后，aastore需要在使用一次，java编译器，最好同dup代替getstatic指令，用于生成之前的每个数组的存贮操作。例如，月份字段。

54.13.5可变参数 可变参数 变长参数函数，实际上使用的就是数组，实际使用的就是数组。

```
public static void f(int... values)
{
    for (int i=0; i<values.length; i++)
        System.out.println(values[i]);
}
public static void main(String[] args)
{
    f (1,2,3,4,5);
}

```

* * *

```
public static void f(int...);
flags: ACC_PUBLIC, ACC_STATIC, ACC_VARARGS
Code:
stack=3, locals=2, args_size=1
0: iconst_0
1: istore_1
2: iload_1
3: aload_0
4: arraylength
5: if_icmpge 23
8: getstatic #2 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
11: aload_0
12: iload_1
13: iaload
14: invokevirtual #3 // Method java/io⤦
Ç /PrintStream.println:(I)V
17: iinc 1, 1
20: goto 2
23: return

```

f()函数，取得一个整数数组，使用的是aload_0 在行偏移3行。取得到了一个数组的大小，等等。

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=4, locals=1, args_size=1
0: iconst_5
1: newarray int
3: dup
4: iconst_0
5: iconst_1
6: iastore
7: dup
8: iconst_1
9: iconst_2
10: iastore
11: dup
12: iconst_2
13: iconst_3
14: iastore
15: dup
16: iconst_3
17: iconst_4
18: iastore
19: dup
20: iconst_4
21: iconst_5
22: iastore
23: invokestatic #4 // Method f:([I)V
26: return

```

素组在main()函数是构造的，使用newarray指令，被填充慢了之后f()被调用。

939 随便提一句，数组对象并不是在main()中销毁的，在整个java中也没有被析构。因为JVM的垃圾收集齐不是自动的，当他感觉需要的时候。 format()方法是做什么的？它用两个参数作为输入，字符串和数组对象。

```
public PrintStream format(String format, Object... args⤦)

```

让我们看一下。

```
public static void main(String[] args)
{
    int i=123;
    double d=123.456;
    System.out.format("int: %d double: %f.%n", i, d⤦Ç );
}

```

* * *

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=7, locals=4, args_size=1
0: bipush 123
2: istore_1
3: ldc2_w #2 // double 123.456⤦
Ç d
6: dstore_2
7: getstatic #4 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
10: ldc #5 // String int: %d⤦
Ç double: %f.%n
12: iconst_2
13: anewarray #6 // class java/⤦
Ç lang/Object
16: dup
17: iconst_0
18: iload_1
19: invokestatic #7 // Method java/⤦
Ç lang/Integer.valueOf:(I)Ljava/lang/Integer;
22: aastore
23: dup
24: iconst_1
25: dload_2
26: invokestatic #8 // Method java/⤦
Ç lang/Double.valueOf:(D)Ljava/lang/Double;
29: aastore
30: invokevirtual #9 // Method java/io⤦
Ç /PrintStream.format:(Ljava/lang/String;[Ljava/lang/Object⤦
Ç ;)Ljava/io/PrintStream;
33: pop
34: return

```

所以int和double类型是被首先普生为integer和double 对象，被用于方法的值。。。format()方法需要，对象雷翔的对象作为输入，因为integer和double类是继承于根类root。他们适合作为数组输入的元素， 另一方面，数组总是同质的，例如，同一个数组不能含有两种不同的数据类型。不能同时都把integer和double类型的数据同时放入的数组。

数组对象的对象在偏移13行，整型对象被添加到在行偏移22. double对象被添加到数组在29行。

倒数第二的pop指令，丢弃了栈顶的元素，因此，这些return执行，堆栈是的空的（平行）

54.13.6 二位数组

二位数组在java 中是一个数组去引用另外一个数组 让我们来创建二位素组。（）

```
public static void main(String[] args)
{
    int[][] a = new int[5][10];
    a[1][2]=3;
}

```

* * *

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=2, args_size=1
0: iconst_5
1: bipush 10
3: multianewarray #2, 2 // class "[[I"
7: astore_1
8: aload_1
9: iconst_1
10: aaload
11: iconst_2
12: iconst_3
13: iastore
14: return

```

它创建使用的是multianewarry指令：对象类型和维数作为操作数，数组的大小（10*5），返回到栈中。（使用iconst_5和bipush指令）

行引用在行偏移10加载（iconst_1和aaload）列引用是选择使用iconst_2指令，在行偏移11行。值得写入和设定在12行，iastore在13 行，写入数据元素？

```
public static int get12 (int[][] in)
{
    return in[1][2];
}

```

* * *

```
public static int get12(int[][]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=2, locals=1, args_size=1
0: aload_0
1: iconst_1
2: aaload
3: iconst_2
4: iaload
5: ireturn

```

引用数组在行2加载，列的设置是在行3，iaload加载数组。

54.13.7 三维数组 三维数组是，引用一维数组引用一维数组。

```
public static void main(String[] args)
{
    int[][][] a = new int[5][10][15];
    a[1][2][3]=4;
    get_elem(a);
}

```

* * *

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=2, args_size=1
0: iconst_5
1: bipush 10
3: bipush 15
5: multianewarray #2, 3 // class "[[[I"
9: astore_1
10: aload_1
11: iconst_1
12: aaload
13: iconst_2
14: aaload
15: iconst_3
16: iconst_4
17: iastore
18: aload_1
19: invokestatic #3 // Method ⤦
Ç get_elem:([[[I)I
22: pop
23: return

```

它是用两个aaload指令去找right引用。

```
public static int get_elem (int[][][] a)
{
    return a[1][2][3];
}

```

* * *

```
public static int get_elem(int[][][]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=2, locals=1, args_size=1
0: aload_0
1: iconst_1
2: aaload
3: iconst_2
4: aaload
5: iconst_3
6: iaload
7: ireturn

```

53.13.8总结

在java中可能出现栈溢出吗？不可能，数组长度实际就代表有多少个对象，数组的边界是可控的，而发生越界访问的情况时，会抛出异常。

54.14 字符串 54.14.1 第一个例子
-----------------------

字符串也是对象，和其他对象的构造方式相同。（还有数组）

```
public static void main(String[] args)
{
    System.out.println("What is your name?");
    String input = System.console().readLine();
    System.out.println("Hello, "+input);
}

```

* * *

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=2, args_size=1
0: getstatic #2 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
3: ldc #3 // String What is⤦
Ç your name?
5: invokevirtual #4 // Method java/io⤦
Ç /PrintStream.println:(Ljava/lang/String;)V
8: invokestatic #5 // Method java/⤦
Ç lang/System.console:()Ljava/io/Console;
11: invokevirtual #6 // Method java/io⤦
Ç /Console.readLine:()Ljava/lang/String;
14: astore_1
15: getstatic #2 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
18: new #7 // class java/⤦
Ç lang/StringBuilder
21: dup
22: invokespecial #8 // Method java/⤦
Ç lang/StringBuilder."<init>":()V
25: ldc #9 // String Hello,
27: invokevirtual #10 // Method java/⤦
Ç lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/⤦
Ç StringBuilder;
30: aload_1
31: invokevirtual #10 // Method java/⤦
Ç lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/⤦
Ç StringBuilder;
34: invokevirtual #11 // Method java/⤦
Ç lang/StringBuilder.toString:()Ljava/lang/String;
37: invokevirtual #4 // Method java/io⤦
Ç /PrintStream.println:(Ljava/lang/String;)V
40: return

```

944 在11行偏移调用了readline()方法，字符串引用（由用户提供）被存储在栈顶，在14行偏移,字符串引用被存储在LVA的1号槽中。

用户输入的字符串在30行偏移处重新加载并和 “hello”字符进行了链接，使用的是StringBulder类，在17行偏移,构造的字符串被pirntln方法打印。

54.14.2 第二个例子 另外一个例子

```
public class strings
{
    public static char test (String a)
    {
        return a.charAt(3);
    };
    public static String concat (String a, String b)
    {
        return a+b;
    }
}

```

* * *

```
public static char test(java.lang.String);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=2, locals=1, args_size=1
0: aload_0
1: iconst_3
2: invokevirtual #2 // Method java/⤦
Ç lang/String.charAt:(I)C
5: ireturn
945

```

字符串的链接使用用StringBuilder类完成。

```
public static java.lang.String concat(java.lang.String, java.⤦
Ç lang.String);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=2, locals=2, args_size=2
0: new #3 // class java/⤦
Ç lang/StringBuilder
3: dup
4: invokespecial #4 // Method java/⤦
Ç lang/StringBuilder."<init>":()V
7: aload_0
8: invokevirtual #5 // Method java/⤦
Ç lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/⤦
Ç StringBuilder;
11: aload_1
12: invokevirtual #5 // Method java/⤦
Ç lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/⤦
Ç StringBuilder;
15: invokevirtual #6 // Method java/⤦
Ç lang/StringBuilder.toString:()Ljava/lang/String;
18: areturn

```

另外一个例子

```
public static void main(String[] args)
{
String s="Hello!";
int n=123;
System.out.println("s=" + s + " n=" + n);
}

```

字符串构造用StringBuilder类，和它的添加方法，被构造的字符串被传递给println方法。

```
public static void main(java.lang.String[]);
flags: ACC_PUBLIC, ACC_STATIC
Code:
stack=3, locals=3, args_size=1
0: ldc #2 // String Hello!
2: astore_1
3: bipush 123
5: istore_2
6: getstatic #3 // Field java/⤦
Ç lang/System.out:Ljava/io/PrintStream;
9: new #4 // class java/⤦
Ç lang/StringBuilder
12: dup
13: invokespecial #5 // Method java/⤦
Ç lang/StringBuilder."<init>":()V
16: ldc #6 // String s=
18: invokevirtual #7 // Method java/⤦
Ç lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/⤦
Ç StringBuilder;
21: aload_1
22: invokevirtual #7 // Method java/⤦
Ç lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/⤦
Ç StringBuilder;
25: ldc #8 // String n=
27: invokevirtual #7 // Method java/⤦
Ç lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/⤦
Ç StringBuilder;
30: iload_2
31: invokevirtual #9 // Method java/⤦
Ç lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
34: invokevirtual #10 // Method java/⤦
Ç lang/StringBuilder.toString:()Ljava/lang/String;
37: invokevirtual #11 // Method java/io⤦
Ç /PrintStream.println:(Ljava/lang/String;)V
40: return
```

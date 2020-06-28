# estools 辅助反混淆 Javascript

0x00 前言
=====

Javascript 作为一种运行在客户端的脚本语言，其源代码对用户来说是完全可见的。但不是每一个 js 开发者都希望自己的代码能被直接阅读，比如恶意软件的制造者们。为了增加代码分析的难度，混淆（obfuscate）工具被应用到了许多恶意软件（如 0day 挂马、跨站攻击等）当中。分析人员为了掀开恶意软件的面纱，首先就得对脚本进行反混淆（deobfuscate）处理。

本文将介绍一些常见的混淆手段和 estools 进行静态代码分析的入门。

0x01 常见混淆手段
=====

加密
---

这类混淆的关键思想在于将需要执行的代码进行一次编码，在执行的时候还原出浏览器可执行的合法的脚本，然后执行之。看上去和可执行文件的加壳有那么点类似。Javascript 提供了将字符串当做代码执行（evaluate）的能力，可以通过[Function 构造器](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function)、[eval](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/eval)、[setTimeout](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/setTimeout)、[setInterval](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/setInterval)将字符串传递给 js 引擎进行解析执行。最常见的是[base62 编码](http://dean.edwards.name/packer/)——其最明显的特征是生成的代码以`eval(function(p,a,c,k,e,r))`开头。

![base62 编码的 Javascript](http://drops.javaweb.org/uploads/images/ec0afbd8e9daeeb19f6b4fdb25056c5ca27a4d20.jpg)

无论代码如何进行变形，其最终都要调用一次 eval 等函数。解密的方法不需要对其算法做任何分析，只需要简单地找到这个最终的调用，改为`console.log`或者其他方式，将程序解码后的结果按照字符串输出即可。自动化的实现方式已经有许多文章介绍过，此处就不再赘述。

隐写术
---

严格说这不能称之为混淆，只是将 js 代码隐藏到了特定的介质当中。如通过最低有效位（LSB）算法嵌入到图片的 RGB 通道、隐藏在图片 EXIF 元数据、隐藏在 HTML 空白字符等。

比如这个耸人听闻的议题：[[一张图片黑掉你：在图片中嵌入恶意程序]](http://www.freebuf.com/news/69106.html)，[PPT](https://conference.hitb.org/hitbsecconf2015ams/wp-content/uploads/2015/02/D1T1-Saumil-Shah-Stegosploit-Hacking-with-Pictures.pdf)放出来一看，正是使用了最低有效位平面算法。结合 HTML5 的 canvas 或者处理二进制数据的 TypeArray，脚本可以抽取出载体中隐藏的数据（如代码）。

![最低有效位](http://drops.javaweb.org/uploads/images/434aed8f339d7a44d5a670fadff1937303fd4b38.jpg)

隐写的方式同样需要解码程序和动态执行，所以破解的方式和前者相同，在浏览器上下文中劫持替换关键函数调用的行为，改为文本输出即可得到载体中隐藏的代码。

复杂化表达式
------

代码混淆不一定会调用 eval，也可以通过在代码中填充无效的指令来增加代码复杂度，极大地降低可读性。Javascript 中存在许多称得上丧心病狂的特性，这些特性组合起来，可以把原本简单的字面量（Literal）、成员访问（MemberExpression）、函数调用（CallExpression）等代码片段变得难以阅读。

Js 中的字面量有字符串、数字、正则表达式

下面简单举一个例子。

*   访问一个对象的成员有两种方法——点运算符和下标运算符。调用 window 的 eval 方法，既可以写成`window.eval()`，也可以`window['eval']`；
    
*   为了让代码更变态一些，混淆器选用第二种写法，然后再在字符串字面量上做文章。先把字符串拆成几个部分：`'e' + 'v' + 'al'`；
    
*   这样看上去还是很明显，再利用一个数字进制转换的技巧：`14..toString(15) + 31..toString(32) + 0xf1.toString(22)`；
    
*   一不做二不休，把数字也展开：`(0b1110).toString(4<<2) + (' '.charCodeAt() - 1).toString(Math.log(0x100000000) / Math.log(2)) + 0xf1.toString(11 << 1)`；
    
*   最后的效果：`window[(2*7).toString(4<<2) + (' '.charCodeAt() - 1).toString(Math.log(0x100000000) / Math.log(2)) + 0xf1.toString(11 << 1)]('alert(1)')`
    

在 js 中可以找到许多这样互逆的运算，通过使用随机生成的方式将其组合使用，可以把简单的表达式无限复杂化。

0x02 静态分析实现
=====

解析和变换代码
-------

本文对 Javascript 实现反混淆的思路是模拟执行代码中可预测结果的部分，编写一个简单的脚本执行引擎，只执行符合某些预定规则的代码块，最后将计算结果替换掉原本冗长的代码，实现表达式的简化。

如果对脚本引擎解释器的原理有初步了解的话，可以知道解释器在为了“读懂”代码，会对源代码进行词法分析、语法分析，将代码的字符串转换为抽象语法树（[Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree), AST）的数据形式。

如这段代码：

`var a = 42; var b = 5; function addA(d) { return a + d; } var c = addA(2) + b;`

对应的语法树如图：

![抽象语法树](http://drops.javaweb.org/uploads/images/b13f1979ebdad9be05fef5aecfc1aae42884cccd.jpg)

（由[JointJS](http://jointjs.com/demos/javascript-ast)的在线工具生成）

不考虑 JIT 技术，解释器可以从语法树的根节点开始，使用深度优先遍历整棵树的所有节点，根据节点上分析出来的指令逐个执行，直到脚本结束返回结果。

通过 js 代码生成抽象语法树的工具很多，如压缩器[UglifyJS](https://github.com/mishoo/UglifyJS)带的 parser，还有本文使用的[esprima](http://esprima.org/)。

esprima 提供的接口很简单：

```
​ var ast = require('esprima').parse(code)

```

另外 Esprima 提供了一个在线工具，可以把任意（合法的）Javascript 代码解析成为 AST 并输出：`http://esprima.org/demo/parse.html`

再结合 estools 的几个辅助库即可对 js 进行静态代码分析：

*   [escope](https://github.com/estools/escope)Javascript 作用域分析工具
    
*   [esutil](https://github.com/estools/esutils)辅助函数库，检查语法树节点是否满足某些条件
    
*   [estraverse](http://github.com/estools/estraverse)语法树遍历辅助库，接口有一点类似 SAX 方式解析 XML
    
*   [esrecurse](http://github.com/estools/esrecurse)另一个语法树遍历工具，使用递归
    
*   [esquery](https://github.com/estools/esquery)使用 css 选择器的语法从语法树中提取符合条件的节点
    
*   [escodegen](http://github.com/estools/escodegen)与 esprima 功能互逆，将语法树还原为代码
    

项目中使用的遍历工具是 estraverse。其提供了两个静态方法，`estraverse.traverse`和`estraverse.replace`。前者单纯遍历 AST 的节点，通过返回值控制是否继续遍历到叶子节点；而 replace 方法则可以在遍历的过程中直接修改 AST，实现代码重构功能。具体的用法可以参考其官方文档，或者本文附带的示例代码。

规则设计
----

从实际遇到的代码入手。最近在研究一些 XSS 蠕虫的时候遇到了类似如下代码混淆：

![代码样本](http://drops.javaweb.org/uploads/images/f706b5405a384ae84f93a71663618c576e74e283.jpg)

观察其代码风格，发现这个混淆器做了这几件事：

*   字符串字面量混淆：首先提取全部的字符串，在全局作用域创建一个字符串数组，同时转义字符增大阅读难度，然后将字符串出现的地方替换成为数组元素的引用
    
*   变量名混淆：不同于压缩器的缩短命名，此处使用了下划线加数字的格式，变量之间区分度很低，相比单个字母更难以阅读
    
*   成员运算符混淆：将点运算符替换为字符串下标形式，然后对字符串进行混淆
    
*   删除多余的空白字符：减小文件体积，这是所有压缩器都会做的事
    

经过搜索，这样的代码很有可能是通过[javascriptobfuscator.com](http://javascriptobfuscator.com/Javascript-Obfuscator.aspx)的免费版生成的。其中免费版可以使用的三个选项（`Encode Strings / Strings / Replace Names`）也印证了前面观察到的现象。

这些变换中，变量名混淆是不可逆的。要是可以智能给变量命名的工具也不错，比如这个[jsnice](http://jsnice.org/)网站提供了一个在线工具，可以分析变量具体作用自动重命名。就算不能做到十全十美，实在不行就用人工的方式，使用 IDE（如 WebStorm）的代码重构功能，结合代码行为分析进行手工重命名还原。

再看字符串的处理。由于字符串将会被提取到一个全局的数组，在语法树中可以观察到这样的特征： 在全局作用域下，出现一个 VariableDeclarator，其 init 属性为 ArrayExpression，而且所有元素都是 Literal ——这说明这个数组所有元素都是常量。简单地将其求值，与变量名（标识符）关联起来。注意，此处为了简化处理，并没有考虑变量名作用域链的问题。在 js 中，作用域链上存在变量名的优先级，比如全局上的变量名是可以被局部变量重新定义的。如果混淆器再变态一点，在不同的作用域上使用相同的变量名，反混淆器又没有处理作用域的情况，将会导致解出来的代码出错。

在测试程序中我设置了如下的替换规则：

*   全局变量声明的字符串数组，在代码中直接使用数字下标引用其值
    
*   结果确定的一连串二元运算，如`1 * 2 + 3 / 4 - 6 % 5`
    
*   正则表达式字面量的 source，字符串字面量的 length
    
*   完全由字符串常量组成的数组，其`join / reverse / slice`等方法的返回值
    
*   字符串常量的`substr / charAt`等方法的返回值
    
*   decodeURIComponent 等全局函数，其所有参数为常量的，替换为其返回值
    
*   结果为常数的数学函数调用，如`Math.sin(3.14)`
    

至于缩进的还原，这是 escodegen 自带的功能。在调用`escodegen.generate`方法生成代码的时候使用默认的配置（忽略第二个参数）即可。

DEMO 程序
-------

这个反混淆器的原型放在 GitHub 上：[https://github.com/ChiChou/etacsufbo](https://github.com/ChiChou/etacsufbo)

运行环境和使用方法参考仓库的 README。

从  [YOU MIGHT NOT NEED JQUERY](http://youmightnotneedjquery.com/)上摘抄了一段代码，放入[javascriptobfuscator.com](https://javascriptobfuscator.com/Javascript-Obfuscator.aspx)测试混淆：

![jsobfuscate.com 混淆样例](http://drops.javaweb.org/uploads/images/5628027112128f9e727c77719eed4b6e56497dd8.jpg)

将混淆结果[https://github.com/ChiChou/etacsufbo/blob/master/tests/cases/jsobfuscator.com.js](https://github.com/ChiChou/etacsufbo/blob/master/tests/cases/jsobfuscator.com.js)进行解开，结果如下：

![6-deobfuscated](http://drops.javaweb.org/uploads/images/4d36030e05cd156482b9d2ea11149d9e11d4094a.jpg)

虽然变量名可读性依旧很差，但已经可以大体看出代码的行为了。

演示程序目前存在大量局限性，只能算一个半自动的辅助工具，还有许多没有实现的功能。

一些混淆器会对字符串字面量进行更复杂的保护，将字符串转换为 f(x) 的形式，其中 f 函数为一个解密函数，参数 x 为密文的字符串。也有原地生成一个匿名函数，返回值为字符串的。这种方式通常使用的函数表达式具有上下文无关的特性——其返回值只与函数的输入有关，与当前代码所处的上下文（比如类的成员、DOM 中取到的值）无关。如以下代码片段中的 xor 函数：

```
var xor = function(str, a, b) {

```

return String.fromCharCode.apply(null, str.split('').map(function(c, i) { var ascii = c.charCodeAt(0); return ascii ^ (i % 2 ? a : b); })); };

如何判断某个函数是否具有这样的特性呢？首先一些库函数可以确定符合，如`btoa，escape，String.fromCharCode`等，只要输入值是常量，返回值就是固定的。建立一个这样的内置函数白名单，接着遍历函数表达式的 AST，若该函数参与计算的参数均没有来自外部上下文，且其所有 CallExpression 的 callee 在函数白名单内，那么通过递归的方式可以确认一个函数是否满足条件。

还有的混淆器会给变量创建大量的引用实例，也就是给同一个对象使用了多个别名，阅读起来非常具有干扰性。可以派出 escope 工具对变量标识符进行数据流分析，替换为所指向的正确值。还有利用数学的恒等式进行混淆的。如声明一个变量 a，若 a 为 Number，则表达式`a-a`、`a * 0`均恒为 0。但如果 a 满足`isNaN(a)`，则表达式返回`NaN`。要清理这类代码，同样需要借助数据流分析的方法。

目前还没有见到使用扁平化流程跳转实现的 js 混淆样本，笔者认为可能跟 js 语言本身的使用场景和特点有关。一般 js 的代都是偏业务型的，不会有太复杂的流程控制或者算法，混淆起来效果不一定理想。

0x03 结束语
=====

Javascript 的确是一门神奇的语言，经常可以遇到一些让人惊讶的奇技淫巧。解密保护过的代码也是有趣的事情。据说几大科技巨头在酝酿给浏览器应用设计一款通用的字节码标准——[WebAssembly](https://github.com/WebAssembly)。一旦这个设想得以实现，代码保护将可以引入真正意义上的“加壳”或者虚拟机保护，对抗技术又将提升到一个新的台阶。

演示项目代码托管在 GitHub：[https://github.com/ChiChou/etacsufbo](https://github.com/ChiChou/etacsufbo)

0x04 参考资料
=====

1.  [http://tobyho.com/2013/12/02/fun-with-esprima/](http://tobyho.com/2013/12/02/fun-with-esprima/)
2.  [https://github.com/estree/estree/blob/master/spec.md](https://github.com/estree/estree/blob/master/spec.md)
3.  [https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey/Parser_API](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey/Parser_API)
4.  [http://jointjs.com/demos/javascript-ast](http://jointjs.com/demos/javascript-ast)
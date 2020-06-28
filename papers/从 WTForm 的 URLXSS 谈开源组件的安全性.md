# 从 WTForm 的 URLXSS 谈开源组件的安全性

0x00 开源组件与开源应用
=====

开源组件是我们大家平时开发的时候必不可少的工具，所谓『不要重复造轮子』的原因也是因为，大量封装好的组件我们在开发中可以直接调用，减少了重复开发的工作量。  
开源组件和开源程序也有一些区别，开源组件面向的使用者是开发者，而开源程序就可以直接面向用户。开源组件，如JavaScript里的uploadify，php里的PHPExcel等；开源程序，如php写的wordpress、joomla，node.js写的ghost等。  
就安全而言，毋庸置疑，开源组件的漏洞影响面远比开源软件要大。但大量开源组件的漏洞却很少出现在我们眼中，我总结了几条原因：

1.  开源程序的漏洞具有通用性，很多可以通过一个通用的poc来测试全网，更具『商业价值』；而开源组件由于开发者使用方法不同，导致测试方法不统一，利用门槛也相对较高
2.  大众更熟悉开源软件，如wordpress，而很少有人知道wordpress内部使用了哪些开源组件。相应的，当出现漏洞的时候人们也只会认为这个漏洞是wordpress的漏洞。
3.  惯性思维让人们认为：『库』里应该不会有漏洞，在代码审计的时候也很少会关注import进来的第三方库的代码缺陷。所以，开源组件爆出的漏洞也较少。
4.  能够开发开源组件的开发者本身素质相对较高，代码质量较高，也使开源组件出漏洞的可能性较小。
5.  组件漏洞多半有争议性，很多锅分不清是组件自身的还是其使用者的，很多问题我们也只能称其为『特性』，但实际上这些特性反而比某些漏洞更可怕。

特别是现在国内浮躁的安全氛围，可以明显感受到第一条原因。就前段时间出现的几个影响较大的漏洞：Java反序列化漏洞、joomla的代码执行、redis的写ssh key，可以明显感觉到后两者炒的比前者要响，而前者不愠不火的，曝光了近一年才受到广泛关注。  
Java反序列化漏洞，恰好就是典型的『组件』特性造成的问题。早在2015年的1月28号，就有白帽子报告了利用Apache Commons Collections这个常用的Java库来实现任意代码执行的方法，但并没有太多关注（原来国外也是这样）。直到11月有人提出了用这个方法攻击WebLogic、WebSphere、JBoss、Jenkins、OpenNMS等应用的时候，才被突然炒起来。  
这种对比明显反应出『开源组件』和『开源应用』在安全漏洞关注度上的差距。  
我个人在乌云上发过几个组件漏洞，从前年发的ThinkPHP框架注入，到后面的Tornado文件读取，到slimphp的XXE，基本都是我自己在使用完这些组件后，对整体代码做code review的时候发现的。  
这篇文章以一个例子，简单地谈谈如何对第三方库进行code review，与如何正确使用第三方库。

0x01 WTForm中的弱validator
=====

WTForms是python web开发中重要的一个组件，它提供了简单的表单生成、验证、转换等功能，是众多python web框架（特别是flask）不可缺少的辅助库之一。  
WTForms中有一个重要的功能就是对用户输入进行检查，在文档中被称为validator：  
http://wtforms.readthedocs.org/en/latest/validators.html

> A validator simply takes an input, verifies it fulfills some criterion, such as a maximum length for a string and returns. Or, if the validation fails, raises a ValidationError. This system is very simple and flexible, and allows you to chain any number of validators on fields.

我们可以简单地使用其内置validator对数据进行检查，比如我们需要用户输入一个『不为空』、『最短10个字符』、『最长64个字符』的『URL地址』，那么我们就可以编写如下class:

```
class MyForm(Form):
    url = StringField("Link", validators=[DataRequired(), Length(min=10, max=64), URL()])

```

以flask为例，在view视图中只需调用validate()函数即可检查用户的输入是否合法：

```
@app.route('/', methods=['POST'])
def check():
    form = MyForm(flask.request.form)
    if form.validate():
        pass # right input
    else:
        pass # bad input

```

典型的敏捷开发手段，减少了大量开发工作量。  
但我自己在做code review的过程中发现，WTForms的内置validators并不可信，与其说是不可信，不如说在安全性上部分validator完全不起任何作用。  
就拿上诉代码为例子，这段代码真的可以检查用户输入的数据是否是一个『URL』么？我们看到wtforms.validators.URL()类：

```
class URL(Regexp):
    def __init__(self, require_tld=True, message=None):
        regex = r'^[a-z]+://(?P<host>[^/:]+)(?P<port>:[0-9]+)?(?P<path>\/.*)?$'
        super(URL, self).__init__(regex, re.IGNORECASE, message)
        self.validate_hostname = HostnameValidation(
            require_tld=require_tld,
            allow_ip=True,
        )

    def __call__(self, form, field):
        message = self.message
        if message is None:
            message = field.gettext('Invalid URL.')

        match = super(URL, self).__call__(form, field, message)
        if not self.validate_hostname(match.group('host')):
            raise ValidationError(message)

```

其继承了Rexexp类，实际上就是对用户输入进行正则匹配。我们看到它的正则：

`regex = r'^[a-z]+://(?P<host>[^/:]+)(?P<port>:[0-9]+)?(?P<path>\/.*)?$'`

可见，这个正则与开发者理解的URL严重的不匹配。大部分的开发者希望获得的URL是一个『HTTP网址』，但这个正则匹配到的却宽泛的太多了，最大特点就是其可匹配任意protocol。  
最容易想到的一个攻击方式就是利用Javascript协议触发的XSS，比如我传入的url是

`javascript://...xss code`

WTForms将认为这是一个合法的URL，并存入数据库。而在业务逻辑中URL通常是输出在超链接的href属性中，而href属性支持利用Javascript伪协议执行JavaScript代码。那么，这里就有极大的可能构造一个XSS攻击。  
另一个草草编写的validator是wtforms.validators.Email()类，查看其代码：

```
class Email(Regexp):
    def __init__(self, message=None):
        self.validate_hostname = HostnameValidation(
            require_tld=True,
        )
        super(Email, self).__init__(r'^.+@([^.@][^@]+)$', re.IGNORECASE, message)

    def __call__(self, form, field):
        message = self.message
        if message is None:
            message = field.gettext('Invalid email address.')

        match = super(Email, self).__call__(form, field, message)
        if not self.validate_hostname(match.group(1)):
            raise ValidationError(message)

```

看看他的正则`^.+@([^.@][^@]+)$`，这个正则根本无法检测用户的输入是否是Email。最前面的.+就让一切坏字符全进入了数据库。  
所以我私下称URL()和Email()为URL Finder和Email Finder，而非validator，因为他们根本无法验证用户输入，倒是更适合作为爬虫查找目标的finder。

0x02 利用弱validator构造XSS
=====

这个漏洞实际上是出现在我写的某个网站中。这个网站允许访客输入其博客地址，而后台使用URL()对地址的合法性进行验证，在用户主页其他用户可以点击其头像访问博客。  
整个过程如下： https://gist.github.com/phith0n/807869afbe1365015627

```
#(๑¯ω¯๑) coding:utf8 (๑¯ω¯๑)
import os
import flask
from flask import Flask
from wtforms.form import Form
from wtforms.validators import DataRequired, URL
from wtforms import StringField
app = Flask(__name__)

class UrlForm(Form):
    url = StringField("Link", validators=[DataRequired(), URL()])

@app.route('/', methods=['GET', 'POST'])
def show_data():
    form = UrlForm(flask.request.form)
    if flask.request.method == "POST" and form.validate():
        url = form.url.data
    else:
        url = flask.request.url
    return flask.render_template('form.html', url=url, form=form)

if __name__ == '__main__':
    app.debug = False
    app.run(os.getenv('IP', '0.0.0.0'), int(os.getenv('PORT', 8080)))

```

form.html:

```
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>test</title>
    </head>
    <body>
        <p>{% if form.url.errors %}
            {{ form.url.errors|join(' ') }}
           {% endif %}
        </p>
        <p>
            your input url
            <a href="{{ url }}" target="_blank">{{ url }}</a>
        </p>
        <form method="post">
            <input type="text" name="url" style="width:300px;" />
            <input type="submit" value="Submit"/>
        </form>
    </body>
</html>

```

demo页面： https://flask-form-phith0n.c9users.io/ 可供测试。  
那么，这段代码存在漏洞吗？回顾URL的正则：

```
regex = r'^[a-z]+://(?P<host>[^/:]+)(?P<port>:[0-9]+)?(?P<path>\/.*)?$'
super(URL, self).__init__(regex, re.IGNORECASE, message)

```

有个//，实在讨厌，将后面的内容全部注释掉了，导致我不能直接执行JavaScript。绕过方法也简单，因为//是单行注释，所以只需换行即可。  
但这里正则修饰符是re.IGNORECASE，并没有re.S，这就导致一旦出现换行这个正则将不再匹配。  
不过这个问题很快也有了答案，在JavaScript中，可以代表换行的字符有\n \r \u2028和\u2029，而在正则里换行仅仅是\n \r，所以我只要通过\u2028或\u2029这两个字符代替换行即可。（\u2028的url编码为%E2%80%A8）  
所以，传入url如下即可：

`javascript://www.baidu.com/ alert(1)`

输入以上url，提交后点击链接即可触发：

![p1](http://drops.javaweb.org/uploads/images/e350193d9f362cfca791d8558130505c83f4108f.jpg)

这个漏洞很典型，任何开发者都不会想到如此平凡的一段代码竟然隐藏着深层次的威胁。  
有些人可能会觉得我这个demo并不能说明实际问题，我简单翻了一下github，不到5分钟就找到了一个存在同样问题的项目： https://github.com/1jingdian/1jingdian 。（虽然其站点已经关闭，但代码可以浏览）

https://github.com/1jingdian/1jingdian/blob/master/application/forms/user.py

```
class SettingsForm(Form):
    motto = StringField('座右铭')
    blog = StringField('博客', validators=[Optional(), URL(message='链接格式不正确')])
    weibo = StringField('微博', validators=[Optional(), URL(message='链接格式不正确')])
    douban = StringField('豆瓣', validators=[Optional(), URL(message='链接格式不正确')])
    zhihu = StringField('知乎', validators=[Optional(), URL(message='链接格式不正确')])

```

这里4个链接，全是用URL()来进行验证。validate()通过后存入数据库。  
之后在个人页面，提取出用户信息传入模板user/profile.html  
https://github.com/1jingdian/1jingdian/blob/master/application/controllers/user.py#L14

```
def profile(uid, page):
    user = User.query.get_or_404(uid)
    votes = user.voted_pieces.paginate(page, 20)
    return render_template('user/profile.html', user=user, votes=votes)

```

跟进一下profile.html  
https://github.com/1jingdian/1jingdian/blob/master/application/templates/user/profile.html

```
{% from "macros/_user.html" import render_user_profile_header %}
...
{{ render_user_profile_header(user, active="votes") }}

```

调用了marco，传入render_user_profile_header函数，继续跟进：  
https://github.com/1jingdian/1jingdian/blob/master/application/templates/macros/_user.html#L37

```
{% macro render_user_profile_header(user, active="creates") %}
   ...
         <div class="media-icons">
            {% if user.blog %}
               <a href="{{ user.blog }}" target="_blank" title="博客">
                  <img src="{{ static('image/media/blog.png') }}" alt=""/>
               </a>
            {% endif %}
            {% if user.weibo %}
               <a href="{{ user.weibo }}" target="_blank" title="微博">
                  <img src="{{ static('image/media/weibo.jpg') }}" alt=""/>
               </a>
            {% endif %}
            {% if user.douban %}
               <a href="{{ user.douban }}" target="_blank" title="豆瓣">
                  <img src="{{ static('image/media/douban.png') }}" alt=""/>
               </a>
            {% endif %}
         </div>
      </div>

```

这里将user.blog、user.weibo、user.douban都放入了a标签的href属性。这一系列操作实际上就是我之前那个demo的缩影，最终导致传入的url过滤不严产生XSS。

0x03 开源组件漏洞到底是谁的锅？
=====

这是屡次受到争议的话题之一，很多人认为开源组件之所以造成了漏洞，都是因为开发者不规范使用组件导致的。  
我觉得认定一个问题是开源组件的锅，那么必须满足以下条件：

*   开发者按照文档常规的方法进行开发
*   文档并没有说明如此开发会存在什么安全问题
*   同样的开发方式在其他同类组件中没有漏洞，而在该组件中产生漏洞

举几个例子，这个漏洞：[WooYun: ThinkPHP某处设计缺陷可导致getshell](http://www.wooyun.org/bugs/wooyun-2015-0101728)。首先满足第一个条件，正常使用S函数。当然文档中也对安全进行了说明：

![p2](http://drops.javaweb.org/uploads/images/b967be6f6028c5d4d4ef173cb276dd2098f692e5.jpg)

但这个说明，我觉得是不够的。你『可以』设置..参数，避免缓存文件名『被猜测到』。文档并没有说明缓存文件名被猜测到有什么危害，也没有强制要求设置这个参数。所以这个锅，官方至少背一半。  
再举个例子：[WooYun: 国际php框架slim架构上存在XXE漏洞（XXE的典型存在形式）](http://www.wooyun.org/bugs/wooyun-2015-0156208)，很明显的一个框架锅，开发者在正常接收POST参数的时候就可以造成XXE漏洞，这个漏洞和开发者是没有任何关系的。  
另一个例子：[WooYun: ThinkPHP架构设计不合理极易导致SQL注入](http://www.wooyun.org/bugs/wooyun-2014-086742)，我们通过修改逻辑运算符改变开发者正常的判断流程，造成安全问题。我们对比一下ThinkPHP和Codeigniter，CI中对于逻辑运算符的位置就和TP不相同，它在『key』的位置：

![p3](http://drops.javaweb.org/uploads/images/901754d293ae88993398bedc27f060c7ef1e9c65.jpg)

正常情况下key位置是不会被用户控制的。所以，同样的开发方式在CI里不存在问题，而在TP里就存在问题，这样的地方我认为也是ThinkPHP的锅。  
我们看本文提出的WTForm的问题，这个锅其实WTForm可以不用独自背。我们在文档中，可以看到它有模模糊糊地提到过validater不严谨的问题：

![p4](http://drops.javaweb.org/uploads/images/b37a628a8f91f33c303043710a97206025fb014f.jpg)

当然，这个模糊的提示对于很多没有安全基础的人来说，很难起到作用。

0x04 开发者如何应对潜在的组件『安全特性』
=====

那么，没有安全基础的开发者，如何去应对潜在的组件安全特性。  
首先，我觉得经常做code review是很有必要的，我会经常把自己写的代码也当做一个开源应用进行阅读与审计，此时会经常发现一些之前没注意到过的安全问题。  
code review的过程中，要深入地跟进一下第三方库的源代码，而不能仅仅是看自己写的代码，这样才能发现一些潜在的特性。这些特性往往是造成漏洞的罪魁祸首。  
另外，文档的阅读能力也是极其重要的一点。其实大量的『框架特性』，框架文档中都有一定的说明。很多开发者更喜欢去看example，觉得看代码比看文字（也许与英文阅读能力也有关系）更直观，而不愿详细阅读说明。这种做法实际上在安全上是非常危险的，因为示例代码通常都是官方给出的最简陋的代码，可能会忽略很多必要的安全措施。  
另外，具备一定的安全基础是每个开发必要的素质，原因不必赘述。
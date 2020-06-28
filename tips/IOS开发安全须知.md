# IOS开发安全须知

最近移动端的漏洞见得比较多，正好从OWASP上找到了IOS开发安全须知，翻译过来，给各位看看。

### 不安全的数据存储 (M1)

毫无疑问，移动设备用户面临的最大风险是设备丢失或被盗。任何捡到或偷盗设备的人都能得到存储在设备上的信息。这很大程度上依赖设备上的应用为存储的数据提供何种保护。苹果的iOS提供了一些机制来保护数据。这些内置的保护措施适合大多数消费级信息。如果要满足更严格的安全需求（如财务数据等），可以在应用程序中内置更好的保护措施。

**补充**

一般来说，一个应用程序应该只存储执行其功能所必须的数据。包括旁路数据在内，如系统日志（见M8章节），无论任何形式的敏感数据，都不应该明文存储在应用的沙箱中（如：~/Documents/*）。消费级的敏感数据应该使用苹果提供的API存储在安全的容器中。

*   少量的消费级敏感数据，如用户身份认证凭证、会话令牌等，可以安全的存储在设备的钥匙扣（Keychain）内（see Keychain Services Reference in Apple’s iOS Developer Library）
*   对于更大或更一般类型的消费级数据，可以安全的使用苹果的文件保护机制存储。(see NSData Class Reference for protection options).
*   如果数据必须要存储在本地，数据的安全敏感度会比普通的消费级敏感度更高，这时可以考虑使用不受苹果的内置加密机制限制（如：keying与用户设备的四位数字PIN编码绑定）的第三方加密API。SQLcipher（http://sqlcipher.net）就是这样一种免费解决方案。在此过程中，适当的密钥管理是极为重要的——当然，这超出了本文的讨论范围。
*   将数据存储在keychain的最安全的API参数是kSecAttrAccessibleWhenUnlocked（在iOS5/6中是默认值）。
*   避免使用NSUserDefaults存储敏感信息。
*   请注意，使用NSManagedObects存储的所有数据/实体都是存放在一个未加密的数据库文件中的。

### 服务端控制薄弱 (M2)

尽管大多数服务器端控制是在服务器端处理的。我们参考Web Service Security Cheat Sheet，其实有些是可以在移动端做的，同时移动端可以帮助服务器做一些必要的工作。

**补充**

设计并实现让移动端和服务端支持的一套共同的安全需求。例如：敏感信息在服务器的处理应该等效于客户端。对所有的客户端输入数据执行积极的输入检查和标准化。使用正则表达式和其他机制来确保只有允许的数据能进入客户端应用程序。如果有可能，对所有的不可信数据进行编码。

### 传输层保护不足 (M3)

网络应用程序的敏感数据被窃听攻击比较常见，iOS手机应用也不例外。

**补充**

所有应用程序都可能在开放的Wi-Fi网络环境中使用，要设计和实现这个场景下的防护措施。列一个清单，确保所有清单内的应用数据在传输过程中得到保护（保护要确保机密性和完整性）。清单中应包括身份认证令牌、会话令牌和应用程序数据。确保传输和接收所有清单数据时使用SSL/TLS加密（See CFNetwork Programming Guide）。确保你的应用程序只接受经过验证的SSL证书（CA链验证在测试环境是禁用的；确保你的应用程序在发布前已经删除这类测试代码）。通过动态测试来验证所有的清单数据在应用程序的操作中都得到了充分保护。通过动态测试，确保伪造、自签名等方式生成的证书在任何情况下都不被应用程序接受。

### 客户端注入(M4)

当移动应用是web应用的时候，数据注入攻击有可能存在，不过攻击场景往往有所不同（例如：利用URL来发送扣费短信或拨打扣费电话）。

**补充**

一般来说，web应用程序的输入验证和输出过滤应该遵循同样的规则。要标准化转换和积极验证所有的输入数据。即使对于本地SQLite/SQLcipher的查询调用，也使用参数化查询。当使用URL scheme时，要格外注意验证和接收输入，因为设备上的任何一个应用程序都可以调用URL scheme。当开发一个web/移动端混合的应用时，保证本地/local的权限是满足其运行要求的最低权限。还有就是控制所有UIWebView的内容和页面，防止用户访问任意的、不可信的网络内容。

### 脆弱的授权和身份认证(M5)

尽管授权和身份认证很大程度上是由服务端来控制的，但是一些移动端特性（如唯一设备标示符）和常见的使用方式也会加剧围绕安全验证、授权用户和其他实体之间的安全问题。

**补充**

一般来说，web应用程序的身份验证和授权应该遵循相同的规则。永远不要使用设备唯一标示符（如UDID、IP、MAC地址、IMEI）来验证用户身份。避免可能的“带外”（out-of-band）身份认证令牌发送到用户用来登陆的相同的设备（如：将短信发送到同一个iPhone）。实现强壮的服务端身份验证、授权和会话管理。验证所有的API请求和支付资源。

### 会话处理不当(M6)

同样，会话处理一般主要是服务器端的工作，但是移动端设备往往有通过不可预见的方式放大传统问题的倾向。例如，在移动端设备上，会话通常要比传统web应用程序的持续时间要长。

**补充**

在大多数情况下，你要遵循与web应用程序相同的会话管理安全实践，两者只有些许不同。永远不用使用设备唯一标示符（如UDID、IP、MAC地址、IEME）来标示一个会话。保证令牌在设备丢失/被盗取、会话被截获时可以被迅速重置。务必保护好认证令牌的机密性和完整性（例如：只使用SSL/TLS来传输数据）。使用可信任的服务来生成会话。

### 通过不可信的输入进行安全决策 (M7)

虽然iOS没有给应用很多彼此通讯的渠道，但存在的那些仍有可能通过数据注入攻击、恶意应用等被攻击者利用。

**补充**

输入验证、输出转义和授权控制相结合可以对付这类缺陷。规范和积极的验证所有输入数据，特别是应用程序之间的边界调用。当使用URL scheme时，要格外小心的验证和接收输入数据，因为设备上的任何应用程序都能调用URL scheme。根据上下文过滤所有不可信的输出，从而保证没有改变应用意图的输出。验证是否允许调用者访问其所请求的资源。如果可能的话，当应用程序访问请求的资源时，提示用户，让用户选择允许/不允许访问。

### 旁道数据泄露 (M8)

旁道数据通常是指那些用来管理或具有非直接功能性目的的I/O数据。如web缓存（用来优化浏览器速度）、击键记录（用户拼写检查）以及其他类似的数据。苹果的iOS提供的一些机制，让一些旁道数据从一个应用程序泄露成为可能。这些数据能够被捡到或偷窃被害人设备的人获取。大多数这类数据都能被应用程序编码控制。

**补充**

在设计和实现所有应用时，都要考虑用户的设备丢失或被盗的情况。首先确认所有的旁道设备数据。这些数据资源至少包括：web缓存、击键记录、屏幕截图、系统日志、剪切缓冲区和第三方类库使用的数据。不要将敏感数据（身份凭证、令牌、个人身份信息PII）放在系统日志中。控制iOS的屏幕截图，防止敏感的应用数据在应用最小化时被截图。在输入敏感数据时，禁用键盘记录，防止这类数据被明文存储到设备上。操作敏感数据时，禁用剪切板缓冲区，防止数据在应用外泄露（被其他应用读取）。动态的测试应用程序的数据存储方式和通讯方式，确保没有敏感信息被不当的传输或存储。

### 失效的密码学 (M9)

尽管绝大多数的加密软件的弱点源于密钥管理安全性不足，但是加密系统的各个方面都应该精心设计和实现。移动应用也是这样。

**补充**

永远不要将密钥硬编码或存储在攻击者可以很简单就能找到的地方：包括明文数据文件、属性文件和编译后的二进制文件。使用安全的容器来存储加密密钥；此外，当密钥是由一个安全服务器控制时，构建一个安全的密钥交换系统，永远不要存储在移动设备上。使用强壮的加密算法及算法实现，包括密钥生成器、哈希等。尽可能使用平台加密API时，如果不能使用，则使用可信任的第三方加密代码。消费级敏感数据应该使用苹果提供的API，将数据存储在安全容器中。  
少量数据，如用户身份认证凭证、会话令牌等，可以安全的存储在设备的Keychain内。(更多请看：Keychain Services Reference in Apple’s iOS Developer Library).

较大或一般类型的数据，苹果的文件保护机制可以保证安全。(更多请看： NSData Class Reference for protection options).

为了更好的保护静态数据，可以使用第三方的加密API，这样就可以不受苹果加密的固有缺陷的限制（如：keying与用户设备的四位数字PIN编码绑定）。SQLcipher是一个免费可用的方案（更多请看：http://sqlcipher.net）。

### 敏感数据泄露 (M10)

各种敏感数据都能被iOS应用程序泄露。更可怕地是，每个应用程序编译后的二进制代码都可以被有能力的对手（攻击者）实施逆向工程。

**补充**

所有必须保证私密的东西都不应放在移动设备上；最好将他们（如算法、专有/机密信息）存储在服务器端。如果私密信息必须存储在移动设备上，尽量将它们保存在进程内存中，如果一定要放在设备存储上，就要做好保护。不要硬编码或简单的存储密码、会话令牌等机密数据。在发布前，清理被编译进二进制数据中的敏感信息，因为编译后的可执行文件仍然可以被逆向破解。  
引用及推荐阅读

OWASP Top 10 Mobile Risks presentation, Appsec USA, Minneapolis, MN, 23 Sept 2011. Jack Mannino, Mike Zusman, and Zach Lanier.  
“iOS Security”, Apple, May 2012, [http://images.apple.com/ipad/business/docs/iOS_Security_May12.pdf](http://images.apple.com/ipad/business/docs/iOS_Security_May12.pdf)

“Deploying iPhone and iPad: Apple Configurator”, Apple, March 2012, [http://images.apple.com/iphone/business/docs/iOS_Apple_Configurator_Mar12.pdf](http://images.apple.com/iphone/business/docs/iOS_Apple_Configurator_Mar12.pdf)

“iPhone OS: Enterprise Deployment Guide”, Apple, 2010, [http://manuals.info.apple.com/en_US/Enterprise_Deployment_Guide.pdf](http://manuals.info.apple.com/en_US/Enterprise_Deployment_Guide.pdf)  
“iPhone in Business”, Apple resources, [http://www.apple.com/iphone/business/resources/](http://www.apple.com/iphone/business/resources/)Apple iOS Developer website.  
"iOS Application (in)Security", MDSec - May 2012, [http://www.mdsec.co.uk/research/iOS_Application_Insecurity_wp_v1.0_final.pdf](http://www.mdsec.co.uk/research/iOS_Application_Insecurity_wp_v1.0_final.pdf)

作者与主编

Ken van Wyk ken[at]krvw.com

翻译

G8dSnowg8dsnow@gmail.com

原文地址：

[https://www.owasp.org/index.php/IOS_Developer_Cheat_Sheet](https://www.owasp.org/index.php/IOS_Developer_Cheat_Sheet)
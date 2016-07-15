---
title: Modern PHP -- 读书笔记
date: 2016-07-14 18:40:28
tags:
    - PHP
categories:
    - Server
---

关于PHP，大家的误解比较多，但其实现代PHP是一门无论开发效率还是执行效率都相当高的编程语言。关于现代PHP的各方面特性，大家可以参考`<Modern PHP>`作者之前写的 **[PHP the right way](http://www.phptherightway.com/)**，中文翻译：**[PHP之道](http://laravel-china.github.io/php-the-right-way/)**。同时，作者也是比较流行的PHP框架 --  **[Slim](http://www.slimframework.com/)** 的开发者。所以这本书非常值得已读，甚至你只需要懂一些OOP的概念即可，并不需要你懂PHP开发。


# Part 1. Language Feature
## Features
### Namespaces
PHP命名空间使用 "\" 字符来分割sumnamespaces。与操作系统的物理文件系统不同，PHP命名空间是一个抽象概念，不必跟文件目录一一对应。大多数PHP Components都是根据PSR-4 autoloader standard来组织subnamespaces与文件目录的映射关系的。

从技术上来说，namespaces仅仅是一个PHP语言的符号，PHP解释器使用这个符号来作为一组classes/interfaces/functions/constants集合的前缀，仅此而已。

`Namespaces are important because they let us create sandboxed code that works alongside other developer's code. This is the cornerstone concept of the modern PHP component ecosystem.`

#### Helpful Tips
**1. Multiple imports**
bad:
```php
<?php
use Symfony\Component\HttpFoundation\Request,
        Symfony\Component\HttpFoundation\Response,
        Symfony\Component\HttpFoundation\Cookie;
```
good:
```php
<?php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpFoundation\Cookie;
```

**2. One class per file**
这点不用多说，每个文件应该只有一个类！

**3.Global namespace**
如果我们引用一个没有命名空间的class/interface/function/constant，PHP会首先假设这个class/interface/function/constant在当前的命名空间中。如果在当前命名空间中没有找到，PHP才会开始resolve。而对于那些没有没有命名空间的代码，PHP认为他们存在于global namespace。

**PSR-4**

### Code to an interface
`An interface is a contract between tow PHP objects that lets one object depend not on what another object is but, instead, on what another can do.`

### Trait

`A trait is a partial class implementation(i.e., constants, properties, and methods) that can be mixed into one or more existing PHP classes. Traits work double duty: they say what a class can do (like an interface), and they provide a modular implementation (like class).`

相对Android开发，我最喜欢的iOS中一个特性就是category，PHP的trait就是有点类似于category，不过还是不太一样的：
1. OC只能针对特定的类进行扩展，而PHP的trait可以将代码单元注入到任意的不相关的类中；
2. 同时OC中的category并不能直接实现属性的扩展，而PHP的trait则能实现常量，属性，以及方法；
3. PHP的trait跟OC的category根本上来说用途是不一样的，OC是对现存类直接扩展，不需要继承实现类。而PHP的trait需要在类的定义中使用 `use` 来明确。

跟class和interface的定义一样，**on trait per file**。

### Generators

`Generators are easy to create because they are just PHP functions that use the yield keyword one or more times. Unlike regular PHP functions, generators never return a value. They only yield values.`

这个概念并不陌生，Python包括Swift都有这个特性，可以用在对大量数据的迭代中，动态去获取数据，而不是一次性生成，避免内存的浪费。

在每次迭代的过程中，PHP都会让Generator实例计算和提供下一个迭代值。在这个过程中，当generator执行到yield value的时候，generator会暂停它的内部状态的执行。只有generator被要求提供下一个迭代值的时候，它才会继续它的内部状态的执行。generator就这样反复pasuing 和 resuming，直到到达generator的函数定义的尾部或empty的时候，generator才会结束执行。

`Generators are a tradeoff between versatility and simplicity. Generators are forward-only iterators.`

### Closures

`A closure is a function that encapsulates its surrounding state at the time it is created. The encapsulated state exists inside the closure even when the closure lives after it original environment ceases to exist.`

这里的闭包是指Closure和Anonymous functions。上面是作者对于闭包的解释，感觉非常准确，比我看到的大多数解释都要简单清晰。闭包在日常业务开发中非常有用，可以非常方便替换我们经常需要用到的delegate设计模式，不需要再去定义一个interface，然后再实现这个interface，再把对应的对象指针传递过去。而是通过Closure，只需要简简单单传递一段代码即可，这个极大简化了日常业务开发。所以目前iOS开发中，大家通常都会使用block来代替delegate设计模式。

PHP Closure or Anonymous function 跟PHP function的定义语法是一样的，但是实际上 Closure 的背后是Closure class的实例，所以Closure被认为是first-class value types。

***Attach State*** : PHP的Closure不会automatically enclose application state，不像JavaScript/OC那样会capture作用域之外的变量。而是，`you must manually attach state to a PHP closure with the closure object's bindTo() method or the use keyword.`

需要注意的是，PHP closures是objects。而我们之所以能让一个closure 实例变量进行调用，是因为这个对象实现 `__invoke()` magic method，当我们在closure实例变量后面跟着一个 `()` 的时候，closure实例变量就会寻找并且调用`__invoke()` 方法，例如 `$closure("Jason")`。

同样，由于PHP closure是objects。所以，在closure内部我们也可以通过 `$this` 访问closure的各种内部状态，但是这个状态是非常boring。同时，closure的`bindTo()`方法可以有一些非常有趣的特殊用法，`This method lets us bind a Closure object's internal state to a different object. The bindTo() method accepts an important second argument that specifies the PHP class of the object to which the closure is bound.This lets the closure access protected and private member variables of the object to which it is bound.`。这个用法有点类似JavaScript的bind方法，可以改变Closure object的 $this 指针指向。

bindTo()这个有趣的用法，经常各种PHP框架的路由所采用，例如：

```php
<? php
class App
{
	protected $routes = array();
    protected $responseStatus = '200 OK';
    protected $responseContentType ='text/html';
    protected $responseBody = 'Hello world';

    public function addRoute($routePath, $routeCallback)
    {
    	// 将Closure bind到App类上
    	$this->routes[$routePath] = $routeCallback->bindTo($this, __CLASS__);
    }

    public function dispatch($currentPath)
    {
    	foreach ($this->routes as $routePath => $callback) {
        	if ($routePath === $currentPath) {
            	$callback();
            }
        }
        // 这里返回的state是在callback内修改过的
        header('HTTP/1.1 '.$this.responseStatus);
        header('Content-type: '.$this.responseContentType);
        header('Content-length: '.mb_strlen($this->responseBody));
        echo $this->responseBody;
    }
}

// 添加注册一个路由
<?php
$app = new App();
$app->addRoute('/users/josh', function() {
	// 因为这个route是bindTo到App class上的，所以这里直接访问$this修改 App 的内部state
	$this->responseContentType = 'application/json;charset=utf8';
    $this->responseBody = '{"name": "Josh"}';
});
$app->dispatch('/users/josh');
```

### Zend Opcache

从PHP 5.5.0开始，PHP引入了内置的bytecode cache支持，叫做 `Zend OPcache`。PHP解释器在执行PHP脚本的时候，会首先把PHP代码编译为Zend Opcodes (machine-code instructions)，然后才会执行bytecode。在所有请求中，PHP解释器都需要这样处理所有的PHP文件，read/parse/compiles，然而我们可以通过把PHP文件预编为PHP bytecode来省略这个开销，这就是`Zend OPcache`。

`Zend OPcache`的使用非常简单，在我们配置之后，它就会在内存中自动缓存precompiled PHP bytecode，在可用的情况就会直接执行这个PHP bytecode，而不需要再去编译PHP代码。

具体配置去google吧，有一点需要注意的是，如果同时配置了 Xdebug的话，在php.ini文件中，需要在Xdebug之前加载Zend OPcache extension扩展。

### Built-in HTTP server

PHP从5.4.0引入了内置的HTTP server，所以我们在不配置Apache或者nginx的情况下就直接预览PHP程序。

`
Remember, the PHP built-in server is a web server. It speaks HTTP, and it can serve static assets in addition to PHP files. It's a great way to write and preview HTML locally without installing MAMP, WAMP,  or a heavyweight web server.
`

要使用内置的HTTP server非常简单，在工程的根目录下，执行下面的命令即可：
```bash
php -S localhost:4000
```
如果要让本地网络的其他设备访问PHP web server，我们可以这么启动：
```bash
php -S 0.0.0.0:4000
```

如果我们希望通过 PHP INI 配置文件去做一些特殊配置，可以通过下面命令来启动：
```bash
php -S localhost:8000 -c app/config/php.ini
```

我们也可以通过Router Scripts来实现一些特殊的路由需求，可以通过下面的命令启动：

```bash
php -S localhost:8000 router.php
```

在PHP代码中，我们可以通过`php_sapi_name()`来判断：

```php
<?php
if (php_sapi_name() === 'cli-server') {
        // PHP web server
} else {
		// Other web server
}
```


# Part 2. Good Pratices

## Standards

### PHP-FIG
PHP-FIG (PHP Framework Interop Group): `The PHP-FIG is a group of PHP framework representatives who, according to the PHP-FIG website, "talk about the commonalities between our projects and find ways we can work together." `

PHP-FIG是由很多不同PHP framework开发者组成的一个开放组织，他们提出的recommendations，不是标准或也不是要求，更像是best pratices的建议集合。不过，目前比较流行大多是PHP框架，比如Laravel或Symfony，都遵守了这些recommendations，所以这个感觉更像是Modern PHP事实上的标准，如果要使用PHP的很多工具和庞大的各种开源库，最好采用这个标准。

`The PHP-FIG's mission is framework interoperability. And framework interoperability means working together via interfaces, autoloading, and style.`
正如下面所说，PHP-FIG的使命就是不同framework之间的互通，让不同框架可以很容易结合在一起使用。而实现互通目前主要通过三个方面来入手：interfaces, autoloading, style：

* Interfaces: `Interfaces enable PHP developers to build, share, and use specialized components instead of monolithic frameworks`，基于interfaces，我们可以做到直接使用某个框架的某个组件，比如Laravel的HTTP的处理部分就是直接使用 Symfony Frameworks的 symfony/httpfoundation 组件，而不用把整个Symfony都集成到Laravel之内。
* Autoloading: `PHP frameworks work together via autoloading. Autoloading is the process by which a PHP class is automatically located and loaded on-demand by the PHP interpreter during runtime`，在autoloading标准出来之前，PHP组件和框架都是基于 `\__autoload()或spl_autoload_register()` 方法来实现自己独特的autoloaders，所以我们要使用一个第三方组件的时候，需要首先去研究一下它的autoloaders的实现。
* Style:  `PHP frameworks work together via code style.`


### PSR
PSR是 `PHP standards recommendation`的缩写，是PHP-FIG提出的recommendations文档，例如PSR-1，PSR-2等。每个PHP-FIG recommendation都是为了解决某个大多数PHP框架开发中常遇到的问题而提出的。

目前在PHP-FIG的官方网站上，[http://www.php-fig.org/psr/](http://www.php-fig.org/psr/) ，可以看到所有的recommendations，目前被采用的有下面几个：

![](/img/PSR.png)

具体的PSR文档内容，可以参考官方网站，PSR-1/2/3/4 几个文档有中文翻译：

文档 | 原文 | 中文翻译
------------- | ------------- | -------------
PSR-1 Basic Coding Standard  | https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-1-basic-coding-standard.md | https://segmentfault.com/a/1190000002521577
PSR-2 Coding Style Guide  | https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md | https://segmentfault.com/a/1190000002521620
PSR-3 Logger Interface | https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md | https://segmentfault.com/a/1190000002521644
PSR-4 AutoLoading Standard | https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md | https://segmentfault.com/a/1190000002521658

#### PSR-1 Basic Coding standard
**PHP tags** : 使用 <?PHP ?> or <?= ?>
**Encoding**: 编码必须使用UTF-8
**Objective**: 单个PHP文件只能定义symbols (a class, trait, function, constant, etc.) 或 perform某种有side effects的action (e.g., create output or manipulate data)。单个文件不能同时包含两种代码
**Autoloading**: PHP namespaces和classes必须支持PSR-4 autoloader standard
**Class names**: PHP类名必须是驼峰命名
**Constant names**: 常量必须使用大写，然后以下划线分割，例如 GREAT_SCOTT
**Method names**: PHP方法命名也必须采用驼峰命名，首字母小写

#### PSR-2 Strict Code Style
**Implement PSR-1** : 要求必须采用PSR-1
**Indentation**: 采用4个空格字符作为缩进
**Files and lines** :  必须使用Unix linefeed(LF)作为结尾；文件最后必须以空行作为结束；不能使用尾部 ?> PHP tag；每行*尽量不要*超过80个字符，*最多不能*超过120个字符；每行结尾不能包含空格；
**Keywords**: 所有的PHP关键字都必须小写
**Namespaces**: 每个namespace声明后面都必须跟着一个空行；使用use来import or alias namespaces的时候，必须在use声明后面跟一个空行；
**Classes**:  定义类时，开始的大括号(opening bracket)必须新起一行，结束的大括号(closing bracket)必须在类体定义后面新起一行；extents/implements关键字必须跟在类名定义的后面；例如：
```php
<?php
namespace My\App;
class Administrator extends User {
    // Class definition body
}
```
**Methods**: 方法定义的大括号规则与类定义类似，opening brackets和closing brackets都必须新起一行。
**Visibility**: 对类中定义的全部property和method都必须声明可见性(visibility)，可见性是public, protected, private其中的一个；abstract / final 必须写在visibility之前；static必须写在visibility之后；
**Control structures** :  所有的control structure keyword (if/elseif/else/switch/case/while/do while/for/foreach/try/catch)的后面都必须一个空字符；大括号的规则与class定义不同，opening brackets跟control structure keyword必须在同一行，而closing bracket必须另新一行；

我们可以通过IDE的格式化工具来让代码格式化，实现PSR-1和PSR-2的code style。我经常使用的工具PHPStorm就可以设置。还有一些其他工具，比如 [PHP-CS-Fixer](http://cs.sensiolabs.org/) 或 [PHP Code Sniffer](http://pear.php.net/manual/en/package.php.php-codesniffer.intro.php)


#### PSR-3 Logger Interface
`PSR-3 is an interface, and it prescribes methods that can be implemented by PHP logger components.`

#### PSR-4 Autoloaders
`An autoloader is a strategy for finding a PHP class, interface, or trait and loading it into the PHP interpreter on-demand at runtime. PHP components and frameworks that support the PSR-4 autoloader standard can be located by and loaded into the PHP interpreter with only one autoloader.`

关于PSR-4，看官方文档之后感觉理解很困惑，本书的作者的解释就非常简洁：
`The essence of PSR-4 is mapping a top-level namespaces prefix to a specific filesystem directory.`，简单来说，就是设定了一个namespaces前缀和某个特定的文件目录之间的映射关系，然后在这个namespace前缀之下如果还有更多的sub namespace，这些sub namespaces就会跟这个特定的目录下面的子目录一一映射起来。例如，\Oreilly\ModernPHP namespace与 src/ 物理路径一一映射，那么\Oreilly\ModernPHP\Chapter1对应的文件夹就是src/Chapter1，而\Oreilly\ModernPHP\Chapter1\Example类对应的文件路径就是src/Chapter1/Example.php文件。

`PSR-4 lets you map a namespace prefix to a filesystem directory. The namespace prefix can be one top-level namespace. The namespace prefix can also be a top-level namespace and any number of subnamespaces. It's quite flexible.`


## Components

### Components
`Modern PHP is less about monolithic framework and more about composing solutions from specialized and interoperable components.`

**What Are Components?**： `A component is a bundle of code that helps solve a specific problem in your PHP application.`

**框架与Components**：如果我们正在创建一个小项目，可以直接使用一些PHP Components集合来解决问题；如果我们正在进行一个多人合作开发的大项目，我们可以通过使用一个Framework；但这都不是绝对的，应该根据具体问题来解决。

**Packagist**：跟其他语言的包管理机制一样，例如Maven，也有一个网站 [https://packagist.org/](https://packagist.org/)  让我们搜索我们需要的PHP Components的相关信息。总所周知的原因，Packagist在国内很不稳定，可以使用国内的全量镜像来代替，[http://www.phpcomposer.com/](http://www.phpcomposer.com/) 。

### Composer

`Composer is a dependency manager for PHP components taht runs on the command line`，跟其他现代语言一样，PHP使用Composer来做依赖管理，类似的有iOS中的Cocoapods，Android中的Maven/gradle，前端的npm，ruby的gem，这些工具可以大大简化我们管理第三方库的成本。于是，当我们在Packagist上面找到我们需要的Components之后，就可以通过Composer来使用这个库。

当我们使用Composer来添加第三方Component的时候，Composer除了会自动帮我们下载需要的PHP Components之外，还会自动帮我们创建一个符合PSR-4的Autoloader。

跟Cocoapods类似，Cocoapods使用Podfile来指定需要依赖的第三方库，以及保存有当前使用的具体的第三方库的版本号。所以我们需要把这两个文件都加入到版本控制中进行管理，确保不同成员/CI/开发环境等不同地方大家使用第三方库版本的一致性。对应Composer中的文件就是 composer.json以及composer.lock。这里需要注意的是composer install 和 composer update 命令的差别：
* composer install，不会安装比composer.lock中列出的更高版本的Components；
* composer update，会更新你的components到最新的稳定版，同时也会更新composer.lock文件为最新的PHP components版本号。

### Semantic Versioning

Modern PHP Components 使用 Semantic Versioning scheme，同时包含了用小数点(.)分隔的三个数字，比如 1.13.2。同时，这个也是很多其他语言开源库的版本规则，对这个一直比较好奇，终于在Modern PHP中看到了相应的解释。

* major release number：第一个数字是major release number，只有当PHP Component发生不再向前兼容的更新时，才需要增加这个版本号。
* minor release number：第二个数字是minor release number，当PHP Component发生一些小的功能更新，并且没有破坏版本兼容时，增加这个版本号。
* patch release number：最后一个数字是patch release number，当发生版本兼容的bug修复的时候，增加这个版本号。

### Create PHP Components

这部分跟iOS创建自己的spec非常相似，并不是非常复杂的问题，参考书或者官方文档很容易就能发布

---
title: Swift实践初探
date: 2016-08-16 13:04:35
tags:
  - iOS
  - Swift
---
# 常用框架

## CocoaPods

### Swift + Pods

Cocoapods从0.36开始通过动态库的形式提供对Swift项目的支持。我们只需要在Podfile中注明，`use_frameworks!`即可，要求项目的最低版本是iOS 8。pods采用动态库的主要原因是因为目前iOS操作系统中并没有自带Swift的运行时支持库，所以所有包含Swift的代码无法被编译为静态库。

参考：

* [https://blog.cocoapods.org/CocoaPods-0.36/](https://blog.cocoapods.org/CocoaPods-0.36/)
* [https://developer.apple.com/library/mac/documentation/MacOSX/Conceptual/BPFrameworks/Frameworks.html](https://developer.apple.com/library/mac/documentation/MacOSX/Conceptual/BPFrameworks/Frameworks.html)

### 问题

大多数第三方库，比如腾讯/微信/微博的SDK都是静态库，虽然这些库提供了Pod的支持，但是在Swift中并没有办法使用。只能把这些第三方库拖入到项目中，然后通过bridge header在项目中使用。比较隐蔽的是腾讯的SDK，虽然也是framework，但是实际上是静态库。

但是不包含静态库的第三方库，比如SDWebImage，却是可以通过Pods直接引用的，Pods会将代码编译为一个动态库，然后集成到App中。在项目中使用的时候，也只需要简简单单通过 `import SDWebImage` 即可，这里面的编译关系都会通过module map进行映射管理。

目前国外的一些比如Fabric SDK，是提供了动态framework的，可以直接在Pods中使用，国内目前我还没有看到。。。leancloud说是提供了，但是打包的时候会坑死你，老老实实用静态库。

参考:

- [https://forum.leancloud.cn/t/sdk-app-store/4516](https://forum.leancloud.cn/t/sdk-app-store/4516)
- [https://forum.leancloud.cn/t/invalid-provisioning-profile/3947](https://forum.leancloud.cn/t/invalid-provisioning-profile/3947)
- [https://fabric.io/kits/ios/crashlytics/install](https://fabric.io/kits/ios/crashlytics/install)

### 关于modules

`Swift’s access control model is based on the concept of modules and source files.`

代码的访问权限控制在开发中是一个非常重要的手段，我们需要隐藏实现细节，只暴露一些特定的interface给特定的调用者，比如Java中的 public/private/protected。在Swift中，访问权限的控制是基于modules和源文件的，public/internal/private。源文件较为容易理解，module则是比较含糊的概念，官方文档中的解释如下：

`A module is a single unit of code distribution—a framework or application that is built and shipped as a single unit and that can be imported by another module with Swift’s import keyword.`

具体来说，对用到Xcode中就是编译target：`Each build target (such as an app bundle or framework) in Xcode is treated as a separate module in Swift.`

而这个module的概念实际上来源于**clang module**，在2012年由Apple工程师提出的概念，用于把传统的C/C++中的include进行简化：`Modules provide an alternative, simpler way to use software libraries that provides better compile-time scalability and eliminates many of the problems inherent to using the C preprocessor to access the API of a library.`

通过这个方式，Swift中引用其他类库的时候，只需要使用import即可，不需要再包含一大堆头文件。如果在一个module内部，编译器会自动处理所有的引用关系，不需要再像OC中引用一大堆繁琐的头文件。在Swift 3.0中，又引入了package manager，跟node/golang中的依赖管理非常类似，但是基本原理应该也是基于module。

参考：

- [http://clang.llvm.org/docs/Modules.html](http://clang.llvm.org/docs/Modules.html)
- [http://andelf.github.io/blog/2014/06/19/modules-for-swift/](http://andelf.github.io/blog/2014/06/19/modules-for-swift/)
- [https://spin.atomicobject.com/2015/02/23/c-libraries-swift/](https://spin.atomicobject.com/2015/02/23/c-libraries-swift/)
- [http://nsomar.com/modular-framework-creating-and-using-them/](http://nsomar.com/modular-framework-creating-and-using-them/)
- [http://adriansampson.net/blog/llvm.html](http://adriansampson.net/blog/llvm.html)
- [http://blog.csdn.net/column/details/xf-llvm.html](http://blog.csdn.net/column/details/xf-llvm.html)
- [http://swift.gg/2016/01/13/swift-ubuntu-x11-window-app/](http://swift.gg/2016/01/13/swift-ubuntu-x11-window-app/)


## 图片

目前开源的较为流行的图片框架包括SDWebImage和Kingfisher，大致对比了一下SDWebImage与Kingfisher的源代码，发现二者的实现思路基本一致，Kingfisher某种程度上来说是SDWebImage的Swift版本。二者共有的特性:

- 通过category机制对业务提供API
- 二级缓存机制，memory缓存使用NSCache；disk缓存使用url MD5之后作为key缓存
- 网络请求使用NSURLSession
- 后台线程进行图片解码
- 所有操作异步化，包括下载，缓存管理，解码
- 支持GIF
- 缓存使用serial queue实现

不同点：

- 下载队列，SDWebImage通过NSOperation进行管理，最大并发请求数为6；Kingfisher没有相关控制
- 图片下载cancel机制，需要手工调用。这个在快速滑动的列表中，是个非常有用的机制。同时对于偶尔某些超大的图片，这个机制也非常有用；请求数量过多会占用过多的端口和系统资源，尽量减少并发的请求数是非常有意义的。
- Kingfisher不支持WebP
- Kingfisher业务开发调用和交互更加友好，配置灵活，例如支持加载完毕的动画效果，延迟加载

## 网络

OC环境下一般都是用AFNetworking，针对Swift，AF的开发者又开发了Alamofire用来取代AF。AF 3.0与Alamofire本身都是基于NSURLSession开发，虽然相对于AF2.0而言代码量减少了很多，但是读起来我自己感觉并不AF2.0结构清晰，AF2.0对runloop和NSOperation的使用，让整个框架变得非常易读。但Alamofire相对AF3.0而言，无论是API还是内部代码，都要更加清晰，这要感谢Swift的语法。比如枚举，extension+protocol都让代码更加容易阅读和维护。

还有一个更重要的地方，Alamofire跟RxSwift的结合非常容易实现，并且会让请求更加容易控制，几乎不需要额外的封装，就可以作为整个项目网络架构存在。

参考：

- [https://github.com/Alamofire/Alamofire](https://github.com/Alamofire/Alamofire)
- [https://github.com/AFNetworking/AFNetworking](https://github.com/AFNetworking/AFNetworking)

## FRP

### RxSwift or RAC?

最开始接触FRP是通过RAC来接触的，但是这个过程相当痛苦，我个人的理解主要原因是RAC并没有完全遵循Rx的理念，发明了Signal这个概念，导致整个学习曲线非常陡。再加上OC的语法限制，当使用非常多高阶函数处理signal时，代码的书写和阅读都非常困难，Swift回归正常的调用语法之后，链式调用让代码读起来感觉舒服很多。

关于ReactiveX，官方网站上的解释是，`ReactiveX is a combination of the best ideas from
the Observer pattern, the Iterator pattern, and functional programming`。ReactiveX是观察者设计模式，迭代器设计模式，和函数式编程的结合，理解这个更加有助于我们更好使用整个框架。如果更多的学习资料和高阶函数的数据流示意可以参考Rx的官方网站：[http://reactivex.io/](http://reactivex.io/)。

关于RX的学习资料，我目前看到最好资料是官方网站的几篇文章和`RxJava Essentials`，这本书上给出的关于FRP的解释我认为是我见过最清晰和容易理解的：`Reactive programming is a programming paradigm based on the concept of an asynchronous data flow. A data flow is like a river: it can be observed, filtered, manipulated, or merged with a second flow to create a new flow for a new consumer.`。Rx为什么好用，看了这本书和官方网站上的文章之后，我觉得有了一个比较直观的解释。简单来说，通过Rx，我们可以使用类似Iterator(例如数组，集合)的操作方式，来处理类似网络请求和UI事件这样的状态，将事件(例如网络请求)的处理简化到跟常量一样容易处理。

相对于RAC，Rx概念更加清晰。data flow类似：**Observables -> Operator -> Observers**，Observables生产实体，Operator进行变换，Observers消费实体。数据流的上游是Observables，中间经过进过Operator变换产生新的Observables，直到到达下游Observers。还有一个比较特殊的存在，**Subject**，既是Observables又是Observers。

具体实现来说，在Swift中，我们使用的时候一般都是通过closure获取数据流，比如onNext()，Observers则被RxSwift实现进行了隐藏，具体实现时这些传入的closure最终会被转换为`AnonymousObserver`：

```Swift
private func reloadData() {
    if disposable != nil {
        disposable?.dispose()
        disposable = nil
    }
    disposable = viewModel
        .updateData()
        .doOnError { [weak self] error in
            JLToast.makeText("网络数据异常，请下拉重试!").show()
            self?.refresher.stopLoad()
        }
        .doOnCompleted { [weak self] in
            self?.refresher.stopLoad()
        }
        .subscribe()
}
```

总结一下，在Swift中，FRP我认为RxSwift是比较占优势的：

- ReactiveX的文档相当全面，并且有各种Operator示意图用于帮助理解
- RxSwift在整体概念上要比RAC清晰不少
- ReactiveX的思想是跨平台的，学习其他语言的基本语法之后，可以做到 "learn once, write everywhere"，这个非常有诱惑力。比如RxJS/RxJava/Rx.NET/Rx.Scala/RxCpp。


### Hot vs Cold & Side Effect

从概念上来说，hot observable与cold observable挺好理解的。我一直比较困惑的是在RxSwift使用的过程中，到底哪些是hot的哪些是cold的。一般来说，我们使用Observable.create创建的Observable，即AnonymousObservable，都是cold observable，只有被subscribe的时候才开始emitting items；hot observable，一个比较典型的例子是Variable，无论是否被subscribe，在value发生变化的时候，都会emitting items。

所以一般来说，我们通过自己创建的Observable来处理状态的时候，面对的都是cold observable。RxSwift中这个概念在实现上没有跟RAC 4.0一样做区分，在使用的过程中，需要根据实际情况注意。

关于Side Effect，简单来说，我们使用Observable.create创建Observable时会传入一个block，这个block在observable被subscribe的时候会被执行，这个就叫做side effect，所以一般来说只有cold observable才会有side effect。这个地方如果不注意，会带来一些意想不到的状况。比如说我们使用Observable封装HTTP请求时，每次这个Observable被subscribe，都会触发一次HTTP请求，所以如果被多次subscribe的话，会导致多次网络请求。无论RAC或者RxSwift都会有这个问题，解决方式也都基本一样，使用[subject](http://reactivex.io/documentation/subject.html)进行multicast。

参考：

- [https://github.com/ReactiveX/RxSwift/blob/master/Documentation/HotAndColdObservables.md](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/HotAndColdObservables.md)
- [http://tech.meituan.com/talk-about-reactivecocoas-cold-signal-and-hot-signal-part-3.html](http://tech.meituan.com/talk-about-reactivecocoas-cold-signal-and-hot-signal-part-3.html)
- [http://www.introtorx.com/content/v1.0.10621.0/14_HotAndColdObservables.html](http://www.introtorx.com/content/v1.0.10621.0/14_HotAndColdObservables.html)
- [https://www.raywenderlich.com/126522/reactivecocoa-vs-rxswift](https://www.raywenderlich.com/126522/reactivecocoa-vs-rxswift)



## 其他框架

### Mantle

OC中常用的框架是Mantle，Mantle的核心代码我之前越过一遍，从工程的角度讲非常不错，可以参考我之前的博客：[http://blog.csdn.net/colorapp/article/details/50277317](http://blog.csdn.net/colorapp/article/details/50277317)。Mantle的实现核心是通过runtime获取属性列表，然后通过 KVC + try/catch 设置value，但是由于Swift是静态语言，这些hack技术都讲无法使用。在Swift中的ORM框架一般都是通过操作符重载来实现ORM过程的简化，我并没有阅读过这种类型框架的源代码，但是就使用感觉而言，并不比Mantle复杂，甚至重载之后的操作符用起来比Mantle中的字典更加安全和可读性更高。

### Masonry

还有另外一个经常使用的框架就是Masonry，之前也阅读过它的源代码，参考之前的博客，[http://blog.csdn.net/colorapp/article/details/45030163](http://blog.csdn.net/colorapp/article/details/45030163)。我认为它的核心是利用Dot Notation实现的链式调用以及巧妙的DSL，极大简化了Auto Layout的开发。Swift中，这个库的作者也开发对应的实现：SnapKit。SnapKit的使用与Masonry的差别并不大，使用起来并不会有任何陌生。源代码暂时还没有阅读，不好对内部的实现进行对比。

无论是Masonry还是SnapKit，都有一个非常需要注意的地方 `update constraints`和`remake constraints`的区别，以及update到底update了什么，什么时候可以用update什么时候不可以。

update constraints首先会比对两个constraints是否相等，如果相等，只更新NSLayoutConstraint的constant字段。不相等会重新添加新的constraints。所以当constraints可能根据交互或者内容的时候，使用constraints需要谨慎，很可能就造成约束冲突。这部分查看Masonry的`MASViewConstraint`的`- (void)install`和`- (MASLayoutConstraint *)layoutConstraintSimilarTo:(MASLayoutConstraint *)layoutConstraint`方法。


# 业务开发

## 源文件

- 在Objective-C开发中，我个人比较推荐单文件单类，因为类的声明是两部分，这样可以让代码的可读性更高，并且可能会造成比如说命名冲突；
- 而在Swift中，代码访问控制更加高级，将多个类组织在一个文件中，代码阅读起来会更加方便一些，并且类名与文件名可以不需要相同
- 同时，可以利用Swift的Module和Pods的Dynamic framework的实现，将项目更合理进行架构划分，让模块关系变得清晰

## Controller

庞大的业务Controller一直是业务开发过程中难以解决的痛点问题。有几种通用的手段：

- MVCS或者MVVM，说一下我自己的理解，两者的共同点都是将数据逻辑从VC中抽取到单独的类中，比如Service或者MVVM，区别只是数据流的方向不一样
- 抽象较为通用的业务逻辑服务化，具体来说就是抽象为较为独立的逻辑为Service，VC直接调用这些Service暴露的接口即可实现某种逻辑，比如账户/分享/定位/数据持久化等
- 基于网络框架提供更高层次的封装，为业务层提供更加简洁的API，例如ORM框架/网络框架的二次封装/Masonry这种UI框架
- 抽取较为通用的View作为项目通用的UI库，例如上下拉组件/Alert组件/ViewPager/下拉框等基本组件，甚至可以解耦发布为pod私有库
- 维护项目中较为通用的工具类，这个在iOS中有个非常有效的技术手段，Objective-C中的category，Swift中的extension。例如：UIImage的基本处理(缩放/截图)/UIView相关/JSON的处理等。这个积累到一定量之后，能大大提高开发效率以及代码的可读性


在Swift中，可以合理利用extension将VC的代码模块化分割，将代码更加合理分布。例如，Eureka的FormViewController的分割：

```Swift
extension FormViewController : UITableViewDataSource {

    //MARK: UITableViewDataSource

    public func numberOfSectionsInTableView(tableView: UITableView) -> Int {
        return form.count
    }

    public func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return form[section].count
    }

    public func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
    	form[indexPath].updateCell()
        return form[indexPath].baseCell
    }

    public func tableView(tableView: UITableView, titleForHeaderInSection section: Int) -> String? {
        return form[section].header?.title
    }

    public func tableView(tableView: UITableView, titleForFooterInSection section: Int) -> String? {
        return form[section].footer?.title
    }
}
```

关于更多的代码实践，由于我目前只是业余时间写了两周Swift，并没有更好的实践可以分享，这部分刚开始，应该来说，best practice还都处于总结中。

## MVVM/MVP/Redux/Protocol-Oriented Programming

在Objective-C开发中，外卖通过广泛使用 MVVM + RAC 来对膨胀的业务逻辑进行控制，效果相当不错，开发效率和代码质量都有不错的提升。得益于Swift的现代语法，我们有了更多的选择，通过这些编程范式，可以让复杂的业务逻辑开发得以简化。MVVM/MVP这里不多说，可以参考的博客有很多，大家应该都有不少接触，下面介绍一下Redux和面向协议编程。

### Redux

FB创造React时提出了Flux数据流概念，意在通过单向数据流和不可变状态简化前端状态的管理，但是FB本身对Flux的实现过于复杂，反而是第三方基于Flux提出的Redux，工程上来说更加简单实用。Redux既可以认为是Flux的具体实现也可以认为是Flux的简化，在实际的 React.js/ReactNative 项目中，大多数都是采用Redux进行日常开发。此外，Redux的文档质量相当高，对前端开发的各种理念都有一些挺不错的总结，可以参考我之前的学习笔记：[Redux学习笔记](http://blog.csdn.net/colorapp/article/details/50256913)。国外的工程师基于Redux实现了Swift版本[ReSwift](https://github.com/ReSwift/ReSwift)。我目前还没有实际中研究过ReSwift，但是就我之前对Redux的研究而言，发现Redux对于客户端开发并不适用，客户端大量的交互对Redux来说是个噩梦，大量的非模态交互，可能导致很多复杂的数据问题，比如要求请求随时可cancel。

### Protocol-Oriented Programming

Protocol-Oriented Programming 是Apple在WWDC 2015发布Swift 2.0的时候推荐的编程范式，可以充分发挥Swift的语法优势，Apple认为这个是Swift的核心。详细大家可以看WWDC上相关的视频，[Protocol-Oriented Programming in Swift](https://developer.apple.com/videos/play/wwdc2015/408/)。这里给大家简单介绍一下，我个人认为当前前端的开发效率的不足，主要的问题是相对后端来说，前端领域的状态比较复杂，很难抽象提取，而Apple推荐面向协议编程也是为了解决iOS开发效率的问题。传统OOP是有缺陷和不足的，下面这些来自 [https://gist.github.com/rbobbins/de5c75cf709f0109ee95](https://gist.github.com/rbobbins/de5c75cf709f0109ee95) :

- Automatic sharing. Two classes can have a reference to the same data, which causes bugs. You try to copy things to stop these bugs, which slows the app down, which causes race conditions, so you add locks, which slows things down more, and then you have deadlock and more complexity. BUGS! This is all due to implicit sharing of mutable state. Swift collections are all value types, so these issues don't happen
- Inheritance is too intrusive. You can only have 1 super class. You end up bloating your super class. Super classes might have stored properties, which bloat it. Initialization is a pain. You don't want to break super class invariants. You have to know what to over ride, and how. This is why we use delegation in Cocoa
- Lost type relationships. You can't count on subclasses to implement some method, e.g:

```swift
class Ordered {
    func precedes(other: Ordered) -> Bool { fatalError("implement me") }
}

class Number: Ordered{
    var value: Int
    override func precedes(other: Ordered) -> Bool {
        return value < other.value //THIS IS THE PROBLEM! WHAT TYPE IS ORDRED?! Does it have a value? Let's force type cast it
    }
}
```

我这里只做一个大概的介绍，详细的资料大家可以参考下面的参考资料。举个让我印象比较深刻的用法，Swift中的 protocol + extension 结合在一起，甚至可以实现类似其他语言中的traits特性 (比如最好的编程语言**PHP**)，将代码的复用颗粒度缩小到某一部分逻辑抽象，不再受限于类继承。这里举个不一定特别合适的例子，比如，飞行这个特性，鸟可以飞行，飞机也能飞行，他们这部分逻辑是能够进行抽象复用的，而鸟和飞机之间却很难抽象到一个基类：

```swift
// 定义protocol
protocol Flyable {
    var speed: Double { get }
}

// 为protocol通过extension添加方法实现
extension Flyable {
    func fly() {
        print("\(self) fly: \(speed) km/h")
    }
}

// 复用飞行逻辑
class Bird: Flyable {
    var speed: Double = 5
}

class Plane: Flyable {
    var speed: Double = 500
}

func play() {
    let bird = Bird()
    bird.fly()
    
    let plane = Plane()
    plane.fly()
}
```
输出：

```
playswift.Bird fly: 5.0 km/h
playswift.Plane fly: 500.0 km/h
```

举个更加实际的例子，错误页面的统一处理，来自 [http://krakendev.io/blog/subclassing-can-suck-and-heres-why](http://krakendev.io/blog/subclassing-can-suck-and-heres-why) ：

```swift
protocol ErrorPopoverRenderer {
    func presentError(message message: String)
}

extension ErrorPopoverRenderer where Self: UIViewController {
    func presentError(message message: String) {
        //Add default implementation here and provide default values to your Error View.
    }
}

class CustomViewController: UIViewController, ErrorPopoverRenderer {
    func failedToEatHuman() {
        //…
        //Throw error because the Kraken sucks at eating Humans today.
        presentError(message: "Oh noes! I didn't get to eat the Human!") //Woohoo! We can provide whatever parameters we want, or no parameters at all!
    }
}
```


参考：

* [https://developer.apple.com/videos/play/wwdc2015/408/](https://developer.apple.com/videos/play/wwdc2015/408/)
* [https://www.raywenderlich.com/109156/introducing-protocol-oriented-programming-in-swift-2](https://www.raywenderlich.com/109156/introducing-protocol-oriented-programming-in-swift-2)
* [http://krakendev.io/blog/subclassing-can-suck-and-heres-why](http://krakendev.io/blog/subclassing-can-suck-and-heres-why)
* [https://gist.github.com/rbobbins/de5c75cf709f0109ee95](https://gist.github.com/rbobbins/de5c75cf709f0109ee95)
* [http://matthewpalmer.net/blog/2015/08/30/protocol-oriented-programming-in-the-real-world/](http://matthewpalmer.net/blog/2015/08/30/protocol-oriented-programming-in-the-real-world/)


# 转变思维

- Protocol
- 泛型
- Extension
- Enum
- Optional
- 错误处理
- Swift的class与NSObject
- 避免OC的黑魔法


参考：

- [http://iosre.com/t/rxswift-runtime-oc-ios/3727](http://iosre.com/t/rxswift-runtime-oc-ios/3727)
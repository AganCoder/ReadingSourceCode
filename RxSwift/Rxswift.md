# RxSwift 源码阅读


仓库地址: https://github.com/ReactiveX/RxSwift/tree/177ab4272f06ee8c14f23cff55021d0468ff2812

## 基本概念介绍

ReactiveX（简写: Rx） 是一个可以帮助我们简化异步编程的框架，RxSwift 是 Rx 的 Swift 版本。RxSwift 结合了观察者模式、迭代器模式和函数响应式编程精华。Rx 不仅仅是一个编程接口，更是一种编程思想上的突破.....

RxSwift 做为 iOS 代表，阅读源码能帮助我们理解作者思想，提高编程能力。 RxSwift 里面内容很多，核心内容分为如下几块：

+ 可观察序列（Observable）
+ 观察者（Observer）
+ 操作符（Operator）
+ 可被清除资源（Disposable）
+ 错误处理（Error Handler）
+ Subject
+ 调度器（Scheduler）

通常而言，`观察者（Observer）`监听`可观察序列（Observable）`的变化做出对应的响应操作。有一些情况下，需要使用 RxSwift 提供的`操作符（Operator）`进行一些特殊的操作，如 Filter 过滤或者 Map 转换等。 观察者与被观察序列生命周期被 `可被清除资源（Disposable）` 管理。多线程相关内容在 `调度器（Scheduler）`中。 

有些内容比较特殊，即可当做`Observer`，同时还可以作为`Observable`，这种就被称为 `Subject`。

也许你会很好奇，一般观察者模式中分为**观察者**与**可观察对象**，怎么在 Rx 中译为**可观察序列**，是不是故弄玄虚？要回答这个问题，我们需要了解 Rx 中的 Event。

```
public enum Event<Element> {
    case next(Element)
    case error(Swift.Error)
    case completed
}
```

+ next(Element):

  ​	Observable 调用 next 发射数据，参数 Element 就是 Observable 发射的数据，这个方法可能会被调用**多次**。

+ error(Swift.Error): 

  ​	当 Observable 出错时调用 error 方法，这个调用会**终止**Observable，后续不会再调用 next 和 completed，参数 Swift.Error 是抛出的异常。

+ completed: 

  ​	正常**终止**，如果没有遇到错误，Observable 在最后一次调用 next 后调用。

next 操作中，Observable 可以发射多次数据，Observable 此时表现就想一个[序列](https://zh.wikipedia.org/wiki/%E5%BA%8F%E5%88%97)

值得一提的是：RxSwift 中，终止只有最后调用 `error(Swift.Error)` 或者 `completed`。

## 从一个简单的例子中说起

```swift
var disposeBag = DisposeBag()

Observable<Int>.create { o in
    DispatchQueue.global().asyncAfter(deadline: .now() + 3.0) {
        o.onNext(1)
        o.onNext(2)
        o.onCompleted()
    }
    return Disposables.create()
}.subscribe { event in
    switch event {
    case .next(let value):
        debugPrint(value)
    case .error(let error):
        debugPrint(error)
    case .completed:
        debugPrint("completed")
    }
}.disposed(by: disposeBag)

```

## 观察者（Observer）与可观察对象（Observable）

## 生命周期管理者 - Disposable 

## 调度器（Scheduler）

## 参考链接

[1. ReactiveX/RxJava文档中文版](https://mcxiaoke.gitbooks.io/rxdocs/content/Subject.html)
[2. RxSwift 中文文档-Disposable - 可被清除的资源](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/disposable.html)
[3. Schedulers - 调度器](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/schedulers.html)
[4. RxSwift核心逻辑（二）-Schedulers](https://www.jianshu.com/p/5bc15220a46c) 
# RxSwift 源码阅读

语言: Swift 
仓库地址: https://github.com/ReactiveX/RxSwift/tree/177ab4272f06ee8c14f23cff55021d0468ff2812

## 简介

ReactiveX（简写: Rx） 是一个可以帮助我们简化异步编程的框架，RxSwift 是 Rx 的 Swift 版本。RxSwift 结合了观察者模式、迭代器模式和函数响应式编程精华。Rx 不仅仅是一个编程接口，更是一种编程思想上的突破.....

RxSwift 做为 iOS 代表，阅读源码能帮助我们理解作者思想，提高编程能力。 RxSwift 里面内容很多，核心内容分为如下几块：

+ 可观察序列（Observable）
+ 观察者（Observer）
+ 主题（Subject）
+ 操作符（Operator）
+ 可被清除资源（Disposable）
+ 调度器（Scheduler）
+ 错误处理（Error Handler）

通常而言，`观察者（Observer）`监听`可观察序列（Observable）`的变化做出对应的响应操作。有一些情况下，需要使用 RxSwift 提供的`操作符（Operator）`进行一些特殊的操作（如 Filter 过滤或者 Map 转换等）。 观察者与被观察序列生命周期被 `可被清除资源（Disposable）` 来管理。多线程相关内容在 `调度器（Scheduler）`中。 

说了这么多，就是想告诉你： RxSwift 内容很多，我们先从一个简单的代码开始看起:

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

在分析这段代码之前，

## 观察者（Observer）与可观察对象（Observable）

## 生命周期管理者 - Disposable 

## 调度器（Scheduler）

## 参考链接

[1. ReactiveX/RxJava文档中文版](https://mcxiaoke.gitbooks.io/rxdocs/content/Subject.html)
[2. RxSwift 中文文档-Disposable - 可被清除的资源](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/disposable.html)
[3. Schedulers - 调度器](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/schedulers.html)
[4. RxSwift核心逻辑（二）-Schedulers](https://www.jianshu.com/p/5bc15220a46c) 

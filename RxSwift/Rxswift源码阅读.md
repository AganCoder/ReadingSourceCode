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
### 创建序列

```swift
extension ObservableType {
    public static func create(_ subscribe: @escaping (AnyObserver<Element>) -> Disposable) -> Observable<Element> {
        AnonymousObservable(subscribe)
    }
}
```

通过协议扩展 `create` 方法，创建了一个 `AnonymousObservable(匿名可观察序列)` 用于保存闭包 `subscribe`

```swift
final private class AnonymousObservable<Element>: Producer<Element> {
    typealias SubscribeHandler = (AnyObserver<Element>) -> Disposable

    let subscribeHandler: SubscribeHandler

    init(_ subscribeHandler: @escaping SubscribeHandler) {
        self.subscribeHandler = subscribeHandler
    }
}
```

类 AnonymousObservable -> 类 Producer -> 类 Observable -> 协议 ObservableType -> 协议 ObservableConvertibleType。

类 Observable 遵守了协议 `ObservableType` 的 `subscribe` 方法，但类 Observable 是一个抽象类型，Producer 类实现了该方法

```swift
public class Observable<Element> : ObservableType {
    public func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
        rxAbstractMethod()
    }
}

class Producer<Element>: Observable<Element> {
    override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
   			// 精简了代码 
    }    
}

```
### 订阅信号

通过协议扩展了 `subscribe` 方法构建匿名观察者 `AnonymousObserver`保存 `on` 的闭包。 然后调用协议 `ObservableType` 的 `subscribe` 方法,也就是最终到类 Producer 的 subscribe 方法。

```swift

// 协议扩展了 subscribe 方法
extension ObservableType {
    public func subscribe(_ on: @escaping (Event<Element>) -> Void) -> Disposable {
        let observer = AnonymousObserver { e in
            on(e)
        }
        return self.asObservable().subscribe(observer)
    }
}

// 匿名观察者，保存着 Subscribe 的闭包
final class AnonymousObserver<Element>: ObserverBase<Element> {
    typealias EventHandler = (Event<Element>) -> Void    
    private let eventHandler : EventHandler
    
    init(_ eventHandler: @escaping EventHandler) {
        self.eventHandler = eventHandler
    }
}
```

再次回顾下:  `AnonymousObservable` 保存了创建`create` 的闭包，而 `AnonymousObserver` 保存着 `subscribe` 的闭包。

在协议扩展了 subscribe 方法中调用 ` self.asObservable().subscribe(observer) ` 方法，会最终调用到  `AnonymousObservable` 的父类 Producer 中。

```swift
class Producer<Element>: Observable<Element> {
    override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
        // 精简了部分代码
        let disposer = SinkDisposer()
        let sinkAndSubscription = self.run(observer, cancel: disposer)
        disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)

        return disposer  
    }  
  
  func run<Observer: ObserverType>(_ observer: Observer, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where Observer.Element == Element {
        rxAbstractMethod()
    }
}

```

 `subscribe` 中调用 `run` 方法， `AnonymousObservable` 重写 `run`  方法


```swift
final private class AnonymousObservable<Element>: Producer<Element> {
    override func run<Observer: ObserverType>(_ observer: Observer, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where Observer.Element == Element {
        let sink = AnonymousObservableSink(observer: observer, cancel: cancel)
        let subscription = sink.run(self)
        return (sink: sink, subscription: subscription)
    }
}
```

`AnonymousObservable` 又引出 `AnonymousObservableSink` 对象，调用了 `AnonymousObservableSink` 的  `run`  方法， `run`  函数内部执行 create 的 闭包。

```swift
final private class AnonymousObservableSink<Observer: ObserverType>: Sink<Observer>, ObserverType {
    typealias Parent = AnonymousObservable<Element>

    override init(observer: Observer, cancel: Cancelable) {
        super.init(observer: observer, cancel: cancel)
    }
		
  	// 精简了代码
    func run(_ parent: Parent) -> Disposable {
        parent.subscribeHandler(AnyObserver(self))
    }
}
```

当订阅后，就会支持 `create` 的闭包，闭包的参数为 `AnyObserver`, 调用 `ObserverType` 的 `func on(_ event: Event<Element>)` 方法通知观察者 `AnonymousObservableSink`（注意不是: `AnonymousObservable`）。

```swift

final private class AnonymousObservableSink<Observer: ObserverType>: Sink<Observer>, ObserverType {
    
    override init(observer: Observer, cancel: Cancelable) {
        super.init(observer: observer, cancel: cancel)
    }

    func on(_ event: Event<Element>) {
        switch event {
        case .next:
            if load(self.isStopped) == 1 {
                return
            }
            self.forwardOn(event)
        case .error, .completed:
            if fetchOr(self.isStopped, 1) == 0 {
                self.forwardOn(event)
                self.dispose()
            }
        }
    }
}


public struct AnyObserver<Element> : ObserverType {

    public init<Observer: ObserverType>(_ observer: Observer) where Observer.Element == Element {
        self.observer = observer.on
    }

    public func on(_ event: Event<Element>) {
        self.observer(event)
    }
}

```

`AnonymousObservableSink` 继承于 `Sink`,  `Sink` 实现了 `final func forwardOn(_ event: Event<Observer.Element>)` 方法，最终会调用到 AnonymousObserver 保存的订阅的闭包里面去。

```swift
class Sink<Observer: ObserverType>: Disposable {
  	final func forwardOn(_ event: Event<Observer.Element>) {
        #if DEBUG
            self.synchronizationTracker.register(synchronizationErrorMessage: .default)
            defer { self.synchronizationTracker.unregister() }
        #endif
        if isFlagSet(self.disposed, 1) {
            return
        }
        self.observer.on(event)
    }
}
```

这就是这个订阅的整个过程，看似简单，其实内部极其复杂。 但是你是否在心中有 N 多个疑问？

### 疑问

#### 1. Sink/AnonymousObservableSink 是干嘛用的？ 

这个[issue](https://github.com/ReactiveX/RxSwift/issues/817) 解释了 Sink 的功能

> It's an internal class used to implement the operators, that receives events and processes them 

如果你仔细阅读其他操作符，你就会明白了.

#### 2. 闭包参数返回值都带了一个 Disposable， Product 的 Subscribe 方法内部返回的 SinkDisposer 都是干嘛用的？ 

Disposable 主要是用于释放资源，这个项目中涉及到两块： Create 以及  Subscribe 。

SinkDisposer 管理 Create 的  Disposable 以及 Sink ，这样主要是为了统一的管理起来， Sink 内部持有 SinkDisposer。

当 complete 或者 error 出现的时候，调用 Sink 的 func dispose() 方法会触发 SinkDisposer 去 func dispose()



#### 3. 怎么保证 `error` 与 `completed` 终止，不在被调用？ 

使用了 `final class AtomicInt: NSLock` 锁，进行比特位置位进行判断处理 



## 参考链接

[1. ReactiveX/RxJava文档中文版](https://mcxiaoke.gitbooks.io/rxdocs/content/Subject.html)
[2. RxSwift 中文文档-Disposable - 可被清除的资源](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/disposable.html)
[3. Schedulers - 调度器](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/rxswift_core/schedulers.html)
[4. RxSwift核心逻辑（二）-Schedulers](https://www.jianshu.com/p/5bc15220a46c) 
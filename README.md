
## RXSwift源码阅读笔记


   * [RXSwift源码阅读笔记](#rxswift源码阅读笔记)
      * [是什么？](#是什么)
      * [RXSwift核心](#rxswift核心)
      * [1. Observable](#1-observable)
         * [1.1 普通序列](#11-普通序列)
         * [1.2 特征序列](#12-特征序列)
            * [1.2.1 PrimitiveSequence特征序列](#121-primitivesequence特征序列)
            * [1.2.2 SharedSequence特征序列](#122-sharedsequence特征序列)
            * [1.2.3 Control特征序列](#123-control特征序列)
      * [2. Observer](#2-observer)
         * [2.1普通观察者](#21普通观察者)
            * [2.1.1 AnyObserver](#211-anyobserver)
            * [2.1.2 AnonymousObserver](#212-anonymousobserver)
            * [2.1.3 subscribe](#213-subscribe)
         * [2.2 特征观察者](#22-特征观察者)
      * [3. Observer &amp; Observable](#3-observer--observable)
         * [3.1 Subject](#31-subject)
         * [3.2 Relay](#32-relay)
      * [4. 操作符](#4-操作符)
      * [5. Disposable](#5-disposable)
         * [5.1 手动管理](#51-手动管理)
         * [5.2 加入销毁包](#52-加入销毁包)
         * [5.3 dispose销毁了什么？](#53-dispose销毁了什么)
            * [5.3.1 Producer类型的序列销毁](#531-producer类型的序列销毁)
            * [5.3.2 Subject 类型的序列销毁](#532-subject-类型的序列销毁)
      * [6. Schedulers](#6-schedulers)
         * [6.1 subscribeOn](#61-subscribeon)
         * [6.2 observeOn](#62-observeon)
         * [6.3 内置的Scheduler](#63-内置的scheduler)



### 是什么？

> ReactiveX is a library for composing **asynchronous（异步）** and **event-based** **programs（基于事件）** by using **observable sequences（可观察序列）**
>
> **RXSwift** 是**ReactiveX** 的 **Swift** 版本，那么我们可以理解成：基于事件和异步组成的可观察序列



### RXSwift核心

> 本文主要对这几个核心通过源码进行展开讲解，描述它们是什么，做了什么事。

* Observable - 可观察序列
* Observer -  观察者
* Observable & Observer 既是可观察序列也是观察者
* Operator \- 操作符，创建变化组合事件
* Disposable - 管理绑定（订阅）的生命周期
* Schedulers \- 线程队列调配







### 1. Observable

> Observable和ObservableType 最重要的就是提供`func subscribe<Observer: ObserverType>`订阅观察者Observer



先来看看`Observable`的源码

```swift
public class Observable<Element> : ObservableType {
    public func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
        rxAbstractMethod()
    }
}
```

`ObservableType`源码

```swift
public protocol ObservableType: ObservableConvertibleType {
    /** Subscribes `observer` to receive events for this sequence. */
    func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element
}
```

`Producer`源码

```swift
class Producer<Element>: Observable<Element> {

    override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
      	... 
        let sinkAndSubscription = self.run(observer, cancel: disposer)
        disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)
        return disposer
    }

    func run<Observer: ObserverType>(_ observer: Observer, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where Observer.Element == Element {
        rxAbstractMethod()
    }
}
```

`Observable.of()`构建一个可观察序列的实现

```swift
let observable = Observable.of("A", "B", "C")

extension ObservableType {
    public static func of(_ elements: Element ..., scheduler: ImmediateSchedulerType = CurrentThreadScheduler.instance) -> Observable<Element> {
        ObservableSequence(elements: elements, scheduler: scheduler)
    }
}

final private class ObservableSequenceSink<Sequence: Swift.Sequence, Observer: ObserverType>: Sink<Observer> where Sequence.Element == Observer.Element {
    ...
    func run() -> Disposable {
        return self.parent.scheduler.scheduleRecursive(self.parent.elements.makeIterator()) { iterator, recurse in
           		...                                                                                  							observer.on(event)
        }
    }
}

final private class ObservableSequence<Sequence: Swift.Sequence>: Producer<Sequence.Element> {
  	...
    init(elements: Sequence, scheduler: ImmediateSchedulerType) {
        self.elements = elements
      	...
    }

    override func run<Observer: ObserverType>(_ observer: Observer, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where Observer.Element == Element {
        let sink = ObservableSequenceSink(parent: self, observer: observer, cancel: cancel)
        let subscription = sink.run()
        return (sink: sink, subscription: subscription)
    }
}
```



可以看出来：

1. `Observable`继承至`ObservableType`并提供了`func subscribe(Observer)`抽象函数
2. `Producer`继承至`Observable`，实现了`func subscribe(Observer)`（主要是调用`func run()`），并提供了`func run<Observer: ObserverType>()`抽象函数
3. **RXSwift**中提供了多种`Observable`的初始化方式，比如上面的`Observable.of()`，而这些初始化方法都继承至`Producer`并提供了`func run<Observer: ObserverType>()`的实现（主要调用`observer.on(event)`发出事件）

总体流程如下：

**Observable**`func subscribe(Observer)`订阅观察者  -> **Producer**`func run()`运行 -> **初始化器**继承**Producer**实现抽象函数`func run()`调用` observer.on(event)`发出事件



总结：

**Observable**提供了订阅观察者的功能，并在内部提供了供订阅时调用的`run`方法来发出事件



#### 1.1 普通序列

> 这些普通序列和`of`类似，都是提供了对`func run<Observer: ObserverType>()`的实现

```swift
// 1.0 empty() 方法 创建一个空内容的 Observable 序列。
	let observable = Observable<Int>.empty()

// 1.1 just()  该方法通过传入一个默认值来初始化。
	let observable = Observable<Int>.just(5)

// 1.2 of() 方法 可变数量的参数
	let observable = Observable.of("A", "B", "C")

// 1.3 from() 方法 需要一个数组参数
	let observable = Observable.from(["A", "B", "C"])

// 1.4 range() 方法 指定起始和结束数值
	let observable = Observable.range(start: 1, count: 5)

// 1.5 generate() 方法 提供判断条件
	let observable = Observable.generate(
   	 initialState: 0,
    	condition: { $0 <= 10 },
    	iterate: { $0 + 2 }
	)

// 1.6 create() 方法 接受一个 block 形式的参数
	let observable = Observable<String>.create{observer in
   		observer.onNext("hangge.com")
   		 return Disposables.create()
	}

// 1.7 deferred() 方法
	相当于是创建一个 Observable 工厂，通过传入一个 block 来执行延迟 Observable序列创建的行为，而这个 block 里就是真正的实例化序列对象的地方。
let factory : Observable<Int> = Observable.deferred {
    isOdd = !isOdd
    if isOdd {
        return Observable.of(1, 3, 5 ,7)
    }else {
        return Observable.of(2, 4, 6, 8)
    }
}

// 1.8 interval() 方法 每隔一段设定的时间，会发出一个索引数的元素
let observable = Observable<Int>.interval(1, scheduler: MainScheduler.instance)
observable.subscribe { event in
    print(event)
}

// 1.9 timer() 方法 延迟执行
	let observable = Observable<Int>.timer(5, scheduler: MainScheduler.instance)

// 延迟定时执行
	let observable = Observable<Int>.timer(5, period: 1, scheduler: MainScheduler.instance)

```



#### 1.2 特征序列

> 特征序列的目的是为了我们更方便使用序列，在普通序列上封装的一层序列，在不同场景选用不同的特征序列



##### 1.2.1 PrimitiveSequence特征序列

single：（**常用在网络请求结果包装**）

* 发出一个元素，或一个 error 事件

Completable：

* 只会发出一个 completed 事件或者一个 error 事件

Maybe：

* 发出一个元素、或者一个 completed 事件、或者一个 error 事件

- - 

```swift
public typealias Single<Element> = PrimitiveSequence<SingleTrait, Element>
public typealias Completable = PrimitiveSequence<CompletableTrait, Swift.Never>
public typealias Maybe<Element> = PrimitiveSequence<MaybeTrait, Element>
```


​	核心代码如下：    

```swift
extension PrimitiveSequenceType where Trait == XXXTrait {
	public static func create(subscribe: @escaping (@escaping SingleObserver) -> Disposable) -> Single<Element> {
    let source = Observable<Element>.create { observer in
        return subscribe { event in
            switch event {
              /**  三个特征序列区别就在此
              single只处理.next()、.error()
              completed只处理.completed()、.error()
              Maybe只处理其中一种就结束
              */
            	observer.on(xxx)
            }
        }
    }
    return PrimitiveSequence(raw: source)
	}

	public func subscribe(_ observer: @escaping (SingleEvent<Element>) -> Void) -> Disposable {
    	...
	}
}
```

总结：

1. `PrimitiveSequence` 初始化方法为**internal**级别，`Single、Completable、Maybe`只能调用`func create()`、`.asSingle()`、`asCompletable()`、`.asMaybe()`生成对象的特征序列
2. `func create()`内部通过封装`Observable<Element>.create() `来创建一个序列





##### 1.2.2 SharedSequence特征序列

```swift
	public typealias Driver<Element> = SharedSequence<DriverSharingStrategy, Element>
	public typealias Signal<Element> = SharedSequence<SignalSharingStrategy, Element>
```

 Driver:

* 主线程监听
* 共享状态变化

 Signal:

* 和Driver唯一不同就是不会发送上一次的事件



主要代码：

```swift
public struct SignalSharingStrategy: SharingStrategyProtocol {
    public static var scheduler: SchedulerType { SharingScheduler.make() }
    
    public static func share<Element>(_ source: Observable<Element>) -> Observable<Element> {
        source.share(scope: .whileConnected)
    }
}
```

```swift
public struct DriverSharingStrategy: SharingStrategyProtocol {
    public static var scheduler: SchedulerType { SharingScheduler.make() }
    public static func share<Element>(_ source: Observable<Element>) -> Observable<Element> {
        source.share(replay: 1, scope: .whileConnected)
    }
}
```

```swift
public enum SharingScheduler {
    /// Default scheduler used in SharedSequence based traits.
    public private(set) static var make: () -> SchedulerType = { MainScheduler() }
}
```

总结：

1. Driver和Signal的实现很简单，就是普通序列调用`share()`实现
2. `SharingScheduler.make() = MainScheduler()`达到了主线程监听的功能
3. Driver和Signal没有提供生成实例的方法，只能通过`func asDriver()`,`func asSignal`转化生成
4. 它们通常用来描述需要共享状态的UI层序列事件







##### 1.2.3 Control特征序列

> 通常用来描述UI控件产生的事件（ControlEvent）和由事件产生的数据变化（ControlProperty）

 ControlProperty：

* 主线程监听
* 订阅时发送上一次的事件
* 不会产生 error 事件

 ControlEvent：

* 主线程监听
* 订阅时不会发送上一次的事件
* 不会产生 error 事件



主要代码：

```swift
public struct ControlProperty<PropertyType> : ControlPropertyType {
    let values: Observable<PropertyType>
    let valueSink: AnyObserver<PropertyType>

    public init<Values: ObservableType, Sink: ObserverType>(values: Values, valueSink: Sink) where Element == Values.Element, Element == Sink.Element {
        self.values = values.subscribe(on: ConcurrentMainScheduler.instance)
        self.valueSink = valueSink.asObserver()
    }
}

public struct ControlEvent<PropertyType> : ControlEventType {
    public init<Ev: ObservableType>(events: Ev) where Ev.Element == Element {
        self.events = events.subscribe(on: ConcurrentMainScheduler.instance)
    }
}
```



```swift
extension Reactive where Base: UIControl {
    public func controlEvent(_ controlEvents: UIControl.Event) -> ControlEvent<()> {
        let source: Observable<Void> = Observable.create { [weak control = self.base] observer in
                MainScheduler.ensureRunningOnMainThread()

                guard let control = control else {
                    observer.on(.completed)
                    return Disposables.create()
                }

                let controlTarget = ControlTarget(control: control, controlEvents: controlEvents) { _ in
                    observer.on(.next(()))
                }

                return Disposables.create(with: controlTarget.dispose)
            }
            .take(until: deallocated)

        return ControlEvent(events: source)
    }

    public func controlProperty<T>(
        editingEvents: UIControl.Event,
        getter: @escaping (Base) -> T,
        setter: @escaping (Base, T) -> Void
    ) -> ControlProperty<T> {
        let source: Observable<T> = Observable.create { [weak weakControl = base] observer in
                guard let control = weakControl else {
                    observer.on(.completed)
                    return Disposables.create()
                }

                observer.on(.next(getter(control)))

                let controlTarget = ControlTarget(control: control, controlEvents: editingEvents) { _ in
                    if let control = weakControl {
                        observer.on(.next(getter(control)))
                    }
                }
                
                return Disposables.create(with: controlTarget.dispose)
            }
            .take(until: deallocated)

        let bindingObserver = Binder(base, binding: setter)

        return ControlProperty<T>(values: source, valueSink: bindingObserver)
    }

    internal func controlPropertyWithDefaultEvents<T>(
        editingEvents: UIControl.Event = [.allEditingEvents, .valueChanged],
        getter: @escaping (Base) -> T,
        setter: @escaping (Base, T) -> Void
        ) -> ControlProperty<T> {
        return controlProperty(
            editingEvents: editingEvents,
            getter: getter,
            setter: setter
        )
    }
}
```



常见的一些使用：

```swift
extension Reactive where Base: UITextField {
    /// 描述事件和事件产生的数据
  	public var text: ControlProperty<String?> {
        return base.rx.controlPropertyWithDefaultEvents(
            getter: { textField in
                textField.text
            },
            setter: { textField, value in
                if textField.text != value {
                    textField.text = value
                }
            }
        )
    }
}

extension Reactive where Base: UIButton {
		/// 描述事件
    public var tap: ControlEvent<Void> {
        controlEvent(.touchUpInside)
    }
}

extension Reactive where Base: UIApplication {
    /// 描述进入后台的通知事件
    public static var didEnterBackground: ControlEvent<Void> {
        let source = NotificationCenter.default.rx.notification(UIApplication.didEnterBackgroundNotification).map { _ in }
        return ControlEvent(events: source)
    }
}
```





### 2. Observer

> ObserverType 最重要的功能就是提供了发送事件的功能`func on(_ event: Event<Element>)`



`ObserverType`源码

```swift
public protocol ObserverType {
    /// The type of elements in sequence that observer can observe.
    associatedtype Element

    /// Notify observer about sequence event.
    ///
    /// - parameter event: Event that occurred.
    func on(_ event: Event<Element>)
}

/// Convenience API extensions to provide alternate next, error, completed events
extension ObserverType {
    
    /// Convenience method equivalent to `on(.next(element: Element))`
    ///
    /// - parameter element: Next element to send to observer(s)
    public func onNext(_ element: Element) {
        self.on(.next(element))
    }
    
    /// Convenience method equivalent to `on(.completed)`
    public func onCompleted() {
        self.on(.completed)
    }
    
    /// Convenience method equivalent to `on(.error(Swift.Error))`
    /// - parameter error: Swift.Error to send to observer(s)
    public func onError(_ error: Swift.Error) {
        self.on(.error(error))
    }
}
```



#### 2.1普通观察者

##### 2.1.1 AnyObserver

```swift
public struct AnyObserver<Element> : ObserverType {
    public init(eventHandler: @escaping EventHandler) {
        self.observer = eventHandler
    }
    
    public func on(_ event: Event<Element>) {
        self.observer(event)
    }
}
```

##### 2.1.2 AnonymousObserver

```swift
final class AnonymousObserver<Element>: ObserverBase<Element> {
    init(_ eventHandler: @escaping EventHandler) {
        self.eventHandler = eventHandler
    }

    override func onCore(_ event: Event<Element>) {
        self.eventHandler(event)
    }
}
```

##### 2.1.3 subscribe

```swift
extension ObservableType {
    
    public func subscribe(_ on: @escaping (Event<Element>) -> Void) -> Disposable {
        let observer = AnonymousObserver { e in
            on(e)
        }
        return self.asObservable().subscribe(observer)
    }
    
    public func subscribe(
        onNext: ((Element) -> Void)? = nil,
        onError: ((Swift.Error) -> Void)? = nil,
        onCompleted: (() -> Void)? = nil,
        onDisposed: (() -> Void)? = nil
    ) -> Disposable {
            let disposable: Disposable
            ...
            let observer = AnonymousObserver<Element> { event in
                switch event {
                case .next(let value):
                    onNext?(value)
                case .error(let error):
                    ...
                case .completed:
                    onCompleted?()
                    disposable.dispose()
                }
            }
            return Disposables.create(
                self.asObservable().subscribe(observer),
                disposable
            )
    }
}
```



总结：

1. `AnyObserver` 和` AnonymousObserver` 的实现没有太大区别
2. `AnyObserver`是`public`，外部可以使用。`AnonymousObserver`为`inner`，只能内部使用。
3. `ObservableType`中提供的`public func subscribe(onNext:...)`实际上就是封装的`AnonymousObserver`.



#### 2.2 特征观察者

> 和特征序列一样，特征观察者的目的是为了更方便的使用它，在普通观察者的基础上封装。



**Binder**

* 不会处理错误事件
* 在主线程上观察



源码:

```swift
public struct Binder<Value>: ObserverType {
    public typealias Element = Value
    
    private let binding: (Event<Value>) -> Void
    public init<Target: AnyObject>(_ target: Target, scheduler: ImmediateSchedulerType = MainScheduler(), binding: @escaping (Target, Value) -> Void) {
        weak var weakTarget = target

        self.binding = { event in
            switch event {
            case .next(let element):
                _ = scheduler.schedule(element) { element in
                    if let target = weakTarget {
                        binding(target, element)
                    }
                    return Disposables.create()
                }
            case .error(let error):
                rxFatalErrorInDebug("Binding error: \(error)")
            case .completed:
                break
            }
        }
    }

    public func on(_ event: Event<Value>) {
        self.binding(event)
    }

    public func asObserver() -> AnyObserver<Value> {
        AnyObserver(eventHandler: self.on)
    }
}
```

总结：

1. Binder 内部封装了事件处理，并且只处理`.next()`， 如果发出一个`.error()`事件则会报错
2. Binder通过AnyObserver包装成一个Observer
3. 特别好用的语法躺



常见的使用：

```swift
let observer: Binder<Bool> = Binder(view) { (view, isHidden) in
    view.isHidden = isHidden
}
```

```swift
extension Reactive where Base: UIView {
  public var isHidden: Binder<Bool> {
      return Binder(self.base) { view, hidden in
          view.isHidden = hidden
      }
  }
}
```



### 3. Observer & Observable

> ObserverType: 发出事件, ObservableType: 订阅观察者
>
> RXSwift和RXRelay中提供了一些特征类型，它们既是Observer也是Observable



#### 3.1 Subject

​	**AsyncSubject**			只发送最后一个信号，并且只在.onCompleted()之后才能接受到

​	**PublishSubject** 		订阅之后才接收元素

​	**ReplaySubject** 		 无论什么时候订阅，发送存储的信号 ，bufferSize确定存储数量

​	**BehaviorSubject**	  存储上一次的信号、初始化时附带一个默认元素（常用）



源码：

```swift
public final class PublishSubject<Element>
    : Observable<Element>
    , SubjectType
    , Cancelable
    , ObserverType
    , SynchronizedUnsubscribeType {
    
    typealias Observers = AnyObserver<Element>.s
    
    /// Creates a subject.
    public override init() {
        super.init()
    }
    
    public func on(_ event: Event<Element>) {
        dispatch(self.synchronized_on(event), event)
    }

    func synchronized_on(_ event: Event<Element>) -> Observers {
	      ...
        switch event {
        case .next:
            return self.observers
        case .completed, .error:
          	...
            return Observers()
        }
    }
    
    public override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
        self.lock.performLocked { self.synchronized_subscribe(observer) }
    }

    func synchronized_subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
        ...
        let key = self.observers.insert(observer.on)
    }
}

```



总结：

1. 继承至Observable，并遵循了ObserverType协议，所以Subject既是Observable，也是Observer.
2. 实现了Observable的`fucn subscribe(Observer)`来订阅观察者，实现了ObserverType的`func on(event）`来发出事件
3. **Subject**与普通观序列不同，它不会继承**Producer**，它自己实现了`subscribe`来处理订阅逻辑
4. `subscribe`订阅时保存了所有的observer，`on`发出事件时取出observers, 分发事件。与**ShareReplay**不同，**Subject**中的所有observer都被订阅，而**ShareReplay**只订阅了第一个观察者，所以**ShareReplay**是共享状态。





#### 3.2 Relay

> 对Subject类型包装，唯一不同的是只发出`onNext()`事件

**PublishRelay** is a wrapper for `PublishSubject` 

**BehaviorRelay** is a wrapper for `BehaviorSubject`

**ReplayRelay** is a wrapper for `ReplaySubject`



最后**ControlProperty**通用也是一个Observer & Observable， 它既可以最为观察者，也可以作为序列





### 4. 操作符

> 处理得到想要的序列

[如何选择操作符](https://beeth0ven.github.io/RxSwift-Chinese-Documentation/content/decision_tree.html)讲的比较详细



### 5. Disposable

> 订阅后的生命周期管理
>
> 管理谁的生命周期？



消耗资源统计

> 需要在Debug环境中定义TRACE_RESOURCES，只需要在Podfile中添加下面的代码

```swift
public class DisposeBase {
    init() {
#if TRACE_RESOURCES
    _ = Resources.incrementTotal()
#endif
    }
    
    deinit {
#if TRACE_RESOURCES
    _ = Resources.decrementTotal()
#endif
    }
}

/// Podfile中添加如下实现
post_install do |installer|
    # Enable tracing resources
    installer.pods_project.targets.each do |target|
      if target.name == 'RxSwift'
        target.build_configurations.each do |config|
          if config.name == 'Debug'
            config.build_settings['OTHER_SWIFT_FLAGS'] ||= ['-D', 'TRACE_RESOURCES']
          end
        end
      end
    end
end

```



生命周期的管理出现在订阅后

```swift
/// 订阅后 -> Disposable
override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable
```



#### 5.1 手动管理

`Disposable.dispose()`销毁

#### 5.2 加入销毁包

`Disposable.disposed(by bag: DisposeBag)`通过`DisposeBag`管理, DisposeBag销毁时会调用`Disposable.dispose()`

```swift
extension Disposable {
    /// Adds `self` to `bag`
    ///
    /// - parameter bag: `DisposeBag` to add `self` to.
    public func disposed(by bag: DisposeBag) {
        bag.insert(self)
    }
}

public final class DisposeBag: DisposeBase {
    
    /// Constructs new empty dispose bag.
    public override init() {
        super.init()
    }

    public func insert(_ disposable: Disposable) {
        self._insert(disposable)?.dispose()
    }
    

    /// This is internal on purpose, take a look at `CompositeDisposable` instead.
    private func dispose() {
        let oldDisposables = self._dispose()

        for disposable in oldDisposables {
            disposable.dispose()
        }
    }
    
    deinit {
        self.dispose()
    }
}
```



#### 5.3 dispose销毁了什么？

##### 5.3.1 Producer类型的序列销毁

来看看**Producer**

```swift
class Producer<Element>: Observable<Element> {
    override init() {
        super.init()
    }

    override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
        return CurrentThreadScheduler.instance.schedule(()) { _ in
            let disposer = SinkDisposer()
            let sinkAndSubscription = self.run(observer, cancel: disposer)
            disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)
            return disposer
        }
    }
}


private final class SinkDisposer: Cancelable {
    func dispose() {
        let previousState = fetchOr(self.state, DisposeState.disposed.rawValue)

        if (previousState & DisposeState.disposed.rawValue) != 0 {
            return
        }

        if (previousState & DisposeState.sinkAndSubscriptionSet.rawValue) != 0 {
            guard let sink = self.sink else {
                rxFatalError("Sink not set")
            }
            guard let subscription = self.subscription else {
                rxFatalError("Subscription not set")
            }

            sink.dispose()
            subscription.dispose()

            self.sink = nil
            self.subscription = nil
        }
    }
}

```

我们最终拿到的`disposer`是`SinkDisposer`, 而我们管理生命周期调用的`dispose`就是将`SinkDisposer`的

```swift
sink.dispose()
subscription.dispose()
self.sink = nil
self.subscription = nil
```

而`sink`和`subscription`来源于`let sinkAndSubscription = self.run(observer, cancel: disposer)`

这时候又回到了我们在**Observable**中提到的`func run()`,它的实现取决于它的使用者：![截屏2022-05-14 上午11.47.51](https://tva1.sinaimg.cn/large/e6c9d24ely1h27sfon3klj20dc0dxgmz.jpg)

我们还是选取**AnonymousObservable**来看看

```swift
final private class AnonymousObservable<Element>: Producer<Element> {
    override func run<Observer: ObserverType>(_ observer: Observer, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where Observer.Element == Element {
        let sink = AnonymousObservableSink(observer: observer, cancel: cancel)
        let subscription = sink.run(self)
        return (sink: sink, subscription: subscription)
    }
}

final private class AnonymousObservableSink<Observer: ObserverType>: Sink<Observer>, ObserverType {
    typealias Element = Observer.Element 
    typealias Parent = AnonymousObservable<Element>

    // state
    private let isStopped = AtomicInt(0)

    #if DEBUG
        private let synchronizationTracker = SynchronizationTracker()
    #endif

    override init(observer: Observer, cancel: Cancelable) {
        super.init(observer: observer, cancel: cancel)
    }

    func on(_ event: Event<Element>) {
        #if DEBUG
            self.synchronizationTracker.register(synchronizationErrorMessage: .default)
            defer { self.synchronizationTracker.unregister() }
        #endif
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

    func run(_ parent: Parent) -> Disposable {
        parent.subscribeHandler(AnyObserver(self))
    }
}

class Sink<Observer: ObserverType>: Disposable {
    func dispose() {
        fetchOr(self.disposed, 1)
        self.cancel.dispose()
    }

    deinit {
#if TRACE_RESOURCES
       _ =  Resources.decrementTotal()
#endif
    }
}
```



**结论**：

我们调用

```swift
override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable
```

得到的`Disposable`就是`AnonymousObservableSink`, 它保存了我们的`Observer`, 我们销毁时就丢弃了`AnonymousObservableSink`信息并执行了`Sink`的`dispose`



##### 5.3.2 Subject 类型的序列销毁

与**Producer**不同，**Subject**订阅重写了订阅方法

```swift
 public override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
      self.lock.performLocked { self.synchronized_subscribe(observer) }
  }

  func synchronized_subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
      if let stoppedEvent = self.stoppedEvent {
          observer.on(stoppedEvent)
          return Disposables.create()
      }

      if self.isDisposed {
          observer.on(.error(RxError.disposed(object: self)))
          return Disposables.create()
      }

      let key = self.observers.insert(observer.on)
      return SubscriptionDisposable(owner: self, key: key)
  }

	func synchronized_unsubscribe(_ disposeKey: DisposeKey) {
        _ = self.observers.removeKey(disposeKey)
  }
```

可以看出来我们订阅观察者后得到了`SubscriptionDisposable(owner: self, key: key)`

```swift
struct SubscriptionDisposable<T: SynchronizedUnsubscribeType> : Disposable {
    private let key: T.DisposeKey
    private weak var owner: T?

    init(owner: T, key: T.DisposeKey) {
        self.owner = owner
        self.key = key
    }

    func dispose() {
        self.owner?.synchronizedUnsubscribe(self.key)
    }
}
```



**结论：**

最终我们要调用的就是`SubscriptionDisposable.dispose()`, 而`self.owner`就是我们保持的**Subject**，最终调用了上面的

```swift
func synchronized_unsubscribe(_ disposeKey: DisposeKey) {
   _ = self.observers.removeKey(disposeKey)
}
```

移除了我们订阅时保存的观察者





### 6. Schedulers

> 描述任务执行在哪个线程，主要用在**subscribeOn**和**observeOn**执行线程的切换，那么我们需要知道：
>
> 1. 哪些任务在**subscribeOn**切换的线程上执行?
> 2. 哪些任务在**observeOn**切换的线程上执行?



#### 6.1 subscribeOn



看看它的源码

```swift
final private class SubscribeOn<Ob: ObservableType>: Producer<Ob.Element> {
    let source: Ob
    let scheduler: ImmediateSchedulerType
    
    init(source: Ob, scheduler: ImmediateSchedulerType) {
        self.source = source
        self.scheduler = scheduler
    }
    
    override func run<Observer: ObserverType>(_ observer: Observer, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where Observer.Element == Ob.Element {
        let sink = SubscribeOnSink(parent: self, observer: observer, cancel: cancel)
        let subscription = sink.run()
        return (sink: sink, subscription: subscription)
    }
}

final private class SubscribeOnSink<Ob: ObservableType, Observer: ObserverType>: Sink<Observer>, ObserverType where Ob.Element == Observer.Element {

	  ...
    func run() -> Disposable {
      	...
        let disposeSchedule = self.parent.scheduler.schedule(()) { _ -> Disposable in
            let subscription = self.parent.source.subscribe(self)
            disposeEverything.disposable = ScheduledDisposable(scheduler: self.parent.scheduler, disposable: subscription)
            return Disposables.create()
        }

        cancelSchedule.setDisposable(disposeSchedule)
        return disposeEverything
    }
}
```



```swift
class Producer<Element>: Observable<Element> {
 

    override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
        ...
        return CurrentThreadScheduler.instance.schedule(()) { _ in
            let disposer = SinkDisposer()
            let sinkAndSubscription = self.run(observer, cancel: disposer)
            disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)

            return disposer
        }
}

```







**总结：**

1. `SubscribeOn`继承至`Producer`，它把序列重新包装成新的`Observable`

2. 在订阅后执行`func run()`, 然后调用我们指定的`scheduler`

   ```swift
   self.parent.scheduler.schedule(()) {
   	let subscription = self.parent.source.subscribe(self)
   }
   ```

   执行后又回到了`source.subscribe(self)`,  这时候就开始执行`Producer`的`subscribe`,而这个订阅就发生在我们指定的`scheduler`中

   ```swift
   CurrentThreadScheduler.instance.schedule(()) { _ in
       let disposer = SinkDisposer()
       let sinkAndSubscription = self.run(observer, cancel: disposer)
       disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)
   
       return disposer
   }
   ```

   



哪些任务在**subscribeOn**切换的线程上执行?

答案：取决于继承`Producer`后，`func run(observer, cancel: disposer)`做的事情



#### 6.2 observeOn



看看源码

```swift
final private class ObserveOnSerialDispatchQueue<Element>: Producer<Element> {
 
		...
    override func run<Observer: ObserverType>(_ observer: Observer, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where Observer.Element == Element {
        let sink = ObserveOnSerialDispatchQueueSink(scheduler: self.scheduler, observer: observer, cancel: cancel)
        let subscription = self.source.subscribe(sink)
        return (sink: sink, subscription: subscription)
    }

}


final private class ObserveOnSerialDispatchQueueSink<Observer: ObserverType>: ObserverBase<Observer.Element> {
    override func onCore(_ event: Event<Element>) {
        _ = self.scheduler.schedule((self, event), action: self.cachedScheduleLambda!)
    }
}
```



**总结：**

1. `ObserveOnSerialDispatchQueue`继承至`Producer`，它把序列重新包装成新的`Observable`

2. 在订阅后执行`func run()`

   ```swift
   let sink = ObserveOnSerialDispatchQueueSink(scheduler: self.scheduler, observer: observer, cancel: cancel)
   let subscription = self.source.subscribe(sink)
   ```

   而这个时候订阅的就是`ObserveOnSerialDispatchQueueSink`,内部重写`onCore()`,  在产生事件就会调用它

   ```swift
   override func onCore(_ event: Event<Element>) {
           _ = self.scheduler.schedule((self, event), action: self.cachedScheduleLambda!)
       }
   ```

   事件在指定的`scheduler`发送



哪些任务在**observeOn**切换的线程上执行?

答案：发送事件和处理事件在切换后的线程执行



#### 6.3 内置的Scheduler

RXSwfit中内置了如下几种 Scheduler：

1. **CurrentThreadScheduler**：表示当前线程 Scheduler。（默认使用这个）
2. **MainScheduler**：表示主线程。如果我们需要执行一些和 UI 相关的任务，就需要切换到该 Scheduler运行。
3. **SerialDispatchQueueScheduler**：封装了 GCD 的串行队列。如果我们需要执行一些串行任务，可以切换到这个 Scheduler 运行。
4. **ConcurrentDispatchQueueScheduler**：封装了 GCD 的并行队列。如果我们需要执行一些并发任务，可以切换到这个 Scheduler 运行。
5. **OperationQueueScheduler**：封装了 NSOperationQueue。



示例：

```swift
let serialQueue =  SerialDispatchQueueScheduler(internalSerialQueueName: "Serial")
let main = MainScheduler()
        
DispatchQueue.global().async {
  Observable<Int>.create { ob in
      print("线程 --- \(Thread.current)")
      ob.onNext(1)
      return Disposables.create {}
	}
	.subscribe(on: main)
	.observe(on: serialQueue)
	.subscribe(onNext: { value in
    	print("线程 --- \(Thread.current)")
	}).disposed(by: self.bag)
}
```

log

```swift
线程 --- <_NSMainThread: 0x60000277c000>{number = 1, name = main}
线程 --- <NSThread: 0x600002761400>{number = 3, name = (null)}
```


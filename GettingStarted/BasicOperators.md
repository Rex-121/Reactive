#BasicOperators



This document explains some of the most common operators used in ReactiveCocoa, and includes examples demonstrating their use.

> 本文档解释了ReactiveCocoa中使用的一些最常见的运算符，并包含演示示例。

Note that “operators”, in this context, refers to functions that transform [`Signal`](../Signal/Signal.md)s and[`SignalProducer`](../SignalProducer/SignalProducer.md)s, *not* custom Swift operators. In other words, these are composable primitives provided by ReactiveCocoa for working with event streams.

> 请注意在此文档中，“运算符”是指转换信号([`Signal`](../Signal/Signal.md))和[`SignalProducer`](../SignalProducer/SignalProducer.md)的函数，而不是自定义的Swift运算符。换句话说，这些是ReactiveCocoa提供的可组合原语，用于处理事件流。

This document will use the term “event stream” when dealing with concepts that apply to both [`Signal`](../Signal/Signal.md) and [`SignalProducer`](../SignalProducer/SignalProducer.md). When the distinction matters, the types will be referred to by name.

> 在处理适用于Signal和SignalProducer的概念时，本文档将使用术语“事件流”。当需要区分二者时，类型将按名称引用。



## Performing side effects with event streams

> 事件流的效应



### Observation

[`Signal`](../Signal/Signal.md)s can be observed with the `observe` function.

> 可以通过`observer()`来观察信号（[`Signal`](../Signal/Signal.md)）

```swift
signal.observe { event in
    switch event {
    case let .value(value):
        print("Value: \(value)")
    case let .failed(error):
        print("Failed: \(error)")
    case .completed:
        print("Completed")
    case .interrupted:
        print("Interrupted")
    }
}
```

Alternatively, callbacks for the `value`, `failed`, `completed` and `interrupted` events can be provided which will be called when a corresponding event occurs.

> 或者可以直接观察值、失败、已完成和中断事件的回调，这些事件将在相应事件发生时被调用。

```swift
signal.observeValues { value in
    print("Value: \(value)")
}

signal.observeFailed { error in
    print("Failed: \(error)")
}

signal.observeCompleted {
    print("Completed")
}

signal.observeInterrupted {
    print("Interrupted")
}
```



### Injecting effects

> 可以使用`on`在事件流上注入副作用，而无需订阅（Observer）它。

```swift
let producer = signalProducer
    .on(starting: { 
        print("Starting")
    }, started: { 
        print("Started")
    }, event: { event in
        print("Event: \(event)")
    }, value: { value in
        print("Value: \(value)")
    }, failed: { error in
        print("Failed: \(error)")
    }, completed: { 
        print("Completed")
    }, interrupted: { 
        print("Interrupted")
    }, terminated: { 
        print("Terminated")
    }, disposed: { 
        print("Disposed")
    })
```

Note that it is not necessary to provide all parameters - all of them are optional, you only need to provide callbacks for the events you care about.

> 请注意，没有必要提供所有参数 - 所有参数都是可选（optional）的，您只需要为您关注的事件提供回调。

Note that nothing will be printed until `producer` is started (possibly somewhere else).

> 请注意，在`prodecer`启动（`start()`）之前不会打印任何内容（你可以在之后的任意时候启动）。



## Operator composition



### Lifting

[`Signal`](../Signal/Signal.md) operators can be *lifted* to operate upon [`SignalProducer`](../SignalProducer/SignalProducer.md) using the `lift` method.

> 信号（[`Signal`](../Signal/Signal.md)）的操作符可以被*`lifted`*，即通过[`SignalProducer`](../SignalProducer/SignalProducer.md) 的`lift()`进行封装。

This will create a new [`SignalProducer`](../SignalProducer/SignalProducer.md) which will apply the given operator to *every* [`Signal`](../Signal/Signal.md) created, just as if the operator had been applied to each produced [`Signal`](../Signal/Signal.md) individually.

> 这将创建一个新的SignalProducer，它将给定的运算符应用于创建的每个Signal上，就像操作符已单独应用于每个生成的Signal一样。



### Transforming event streams

These operators transform an event stream into a new stream.

> 以下运算符将事件流转换为新事件流。



### Mapping

The `map` operator is used to transform the values in an event stream, creating a new stream with the results.

> `map`用于转换事件流中的值，并使用结果创建新流。

```swift
let (signal, observer) = Signal<String, NoError>.pipe()

signal
    .map { string in string.uppercased() }
    .observeValues { value in print(value) }

observer.send(value: "a")     // Prints A
observer.send(value: "b")     // Prints B
observer.send(value: "c")     // Prints C
```

[Interactive visualisation of the `map` operator.](http://neilpa.me/rac-marbles/#map)



### Filtering

The `filter` operator is used to only include values in an event stream that satisfy a predicate.

> `filter`将过滤掉事件流中不满足谓词的值。

```swift
let (signal, observer) = Signal<Int, NoError>.pipe()

signal
    .filter { number in number % 2 == 0 }
    .observeValues { value in print(value) }

observer.send(value: 1)     // Not printed
observer.send(value: 2)     // Prints 2
observer.send(value: 3)     // Not printed
observer.send(value: 4)     // prints 4
```



### Aggregating

The `reduce` operator is used to aggregate a event stream’s values into a single combined value. Note that the final value is only sent after the input stream completes.

> `reduce`用于将事件流的值聚合为单个组合值。请注意，最终值仅在输入流完成后发送。

```swift
let (signal, observer) = Signal<Int, NoError>.pipe()

signal
    .reduce(1) { $0 * $1 }
    .observeValues { value in print(value) }

observer.send(value: 1)     // nothing printed
observer.send(value: 2)     // nothing printed
observer.send(value: 3)     // nothing printed
observer.sendCompleted()   // prints 6
```

The `collect` operator is used to aggregate a event stream’s values into a single array value. Note that the final value is only sent after the input stream completes.
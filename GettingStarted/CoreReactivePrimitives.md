# Core Reactive Primitives

1. [`Signal`](#signal-a-unidirectional-stream-of-events)

1. [`Event`](#event-the-basic-transfer-unit-of-an-event-stream)

1. [SIgnalProducer](#SignalProducer: deferred work that creates a stream of values.)

1. [Property](#Property: an observable box that always holds a value.)

1. [Action](#Action: a serialized worker with a preset action.)

1. [`Lifetime`](#lifetime-limits-the-scope-of-an-observation)

### `Signal`: a unidirectional stream of events.

> 信号: 单方向的事件流

The owner of a ```Signal``` has unilateral control of the event stream. ```Observers``` may register their interests in the future events at any time, but the observation would have no side effect on the stream or its owner.

> 信号的所有者可以单方面控制事件流。观察者(Observers)可以在未来的任何时候注册并成为此事件的观察者，并且观察者对事件流或其所有者没有任何副作用。

It is like a live TV feed — you can observe and react to the content, but you cannot have a side effect on the live feed or the TV station.

> 它就像一个直播节目 - 你可以收看节目，且你不会对现场直播或电视台产生副作用。

```swift
let channel: Signal<Program, NoError> = tvStation.channelOne
channel.observeValues { program in ... }
```

See also: The ```Signal``` overview, The ```Signal``` contract, The ```Signal``` API reference

### `Event`: the basic transfer unit of an event stream.

> 事件：事件流的基本传输单元。

A ```Signal``` may have any arbitrary number of events carrying a value, following by an eventual terminal event of a specific reason.

> 信号可以拥有任意数量的带值事件，衍生成特定值最终事件。

It is like a frame in a one-time live feed — seas of data frames carry the visual and audio data, but the feed would eventually be terminated with a special frame to indicate end of stream.

> 它就像一次性的实时帧 - 视觉数据加上音频数据最终通过终端事件处理为播放数据。

See also: The ```Event``` overview, The ```Event``` contract, The ```Event``` API reference

### `SignalProducer`: deferred work that creates a stream of values.

> SignalProducer：延迟创建值流。

`SignalProducer` defers work — of which the output is represented as a stream of values — until it is started. For every invocation to start the SignalProducer, a new Signal is created and the deferred work is subsequently invoked.

> SignalProducer 当它被启动时才会创建（输出为值流）。对于每次启动SignalProducer的调用，都会创建一个新的Signal，并随后调用延迟的工作。

It is like a on-demand streaming service — even though the episode is streamed like a live TV feed, you can choose what you watch, when to start watching and when to interrupt it.

> 它就像一个点播流媒体服务 - 即使这一集是直播节目（直播流），你仍可以选择何时观看或关闭你点播的节目。

```swift
let frames: SignalProducer<VideoFrame, ConnectionError> = vidStreamer.streamAsset(id: tvShowId)
let interrupter = frames.start { frame in ... }
interrupter.dispose()
```

See also: The ```SignalProducer``` overview, The ```SignalProducer``` contract, The ```SignalProducer``` API reference

### `Property`: an observable box that always holds a value.

> 属性：一个始终有值的观察盒。

```Property``` is a variable that can be observed for its changes. In other words, it is a stream of values with a stronger guarantee than ```Signal``` — the latest value is always available, and the stream would never fail.

> 属性是可以观察其变化的变量。换句话说，它是一个具有比Signal更强保证的值流 - 会一直提供一个可用的不会失败的新值。

It is like the continuously updated current time offset of a video playback — the playback is always at a certain time offset at any time, and it would be updated by the playback logic as the playback continues.

> 这就像视频的播放时间会不断向前偏移 - 播放总是在任何时间的某个时间偏移，并且当播放继续时它将由播放逻辑更新。

```swift
let currentTime: Property<TimeInterval> = video.currentTime
print("Current time offset: \(currentTime.value)")
currentTime.signal.observeValues { timeBar.timeLabel.text = "\($0)" }
```

See also: The ```Property``` overview, The ```Property``` contract, The ```Property``` API reference

### `Action`: a serialized worker with a preset action.

> 操作：具有预设操作的序列化事件。

When being invoked with an input, Action apply the input and the latest state to the preset action, and pushes the output to any interested parties.

> 使用带有输入值(Input)的操作(Action)时，操作(Action)会捕获输入的值(Input)和最新状态应用于预设操作，并将输出推送给任何感兴趣的观察者。

It is like an automatic vending machine — after choosing an option with coins inserted, the machine would process the order and eventually output your wanted snack. Notice that the entire process is mutually exclusive — you cannot have the machine to serve two customers concurrently.

> 它就像一台自动售货机 - 在选择你要的货物并塞入硬币后，机器将处理订单并"输出"您想要的零食。请注意，整个过程是互斥的 - 您不能让机器同时为两个客户提供服务。

```swift
// Purchase from the vending machine with a specific option.
vendingMachine.purchase
.apply(snackId)
.startWithResult { result
switch result {
case let .success(snack):
print("Snack: \(snack)")

case let .failure(error):
// Out of stock? Insufficient fund?
print("Transaction aborted: \(error)")
}
}

// The vending machine.
class VendingMachine {
let purchase: Action<Int, Snack, VendingMachineError>
let coins: MutableProperty<Int>

// The vending machine is connected with a sales recorder.
init(_ salesRecorder: SalesRecorder) {
coins = MutableProperty(0)
purchase = Action(state: coins, enabledIf: { $0 > 0 }) { coins, snackId in
return SignalProducer { observer, _ in
// The sales magic happens here.
// Fetch a snack based on its id
}
}

// The sales recorders are notified for any successful sales.
purchase.values.observeValues(salesRecorder.record)
}
}
```

See also: The ```Action``` overview, The ```Action``` API reference

### Lifetime: limits the scope of an observation

> 寿命：限制观察的范围

When observing a ```Signal``` or ```SignalProducer```, it doesn’t make sense to continue emitting values if there’s no longer anyone observing them. Consider the video stream: once you stop watching the video, the stream can be automatically closed by providing a ```Lifetime```:

> 观察Signal或SignalProducer时，如果没有人观察它们，继续提供值是没有意义的。以视频流为例：一旦您停止观看视频，拥有寿命(Lifetime)的流可以自动关闭：

```swift
class VideoPlayer {
private let (lifetime, token) = Lifetime.make()

func play() {
let frames: SignalProducer<VideoFrame, ConnectionError> = ...
frames.take(during: lifetime).start { frame in ... }
}
}
```

See also: The ```Lifetime``` overview, The ```Lifetime``` API reference

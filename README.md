# HackingCombine
Exploring uncharted territory in Combine. 🤠

## What is Combine?

If you're asking this question, maybe you should head over to [Combine101](https://github.com/learncombine/Combine101) first?

## How can you pause and resume a subscription?

Publishers and subscribers have this basic interplay:

- A subscriber requests to subscribe
- A publisher gives the subscriber a subscription
- A subscriber requests values
- A publisher sends values
- A publisher sends a completion

It's when the publisher sends values to the subscriber, or more specifcally, the subscriber _receives_ those values, that it can _adjust_ its demand for future values (additively).

Let's start this example off by creating a custom subscriber:

```swift
final class IntSubscriber: Subscriber {
    typealias Input = Int
    typealias Failure = Never
    
    // 1
    var subscription: Subscription?
    
    func receive(subscription: Subscription) {
        // 2
        subscription.request(.max(2))
        self.subscription = subscription
    }
    
    func receive(_ input: Int) -> Subscribers.Demand {
        print("Received value", input)
        switch input {
        case 1:
            return .max(1) // 3
        default:
            return .none // 4
        }
    }
    
    func receive(completion: Subscribers.Completion<Never>) {
        print("Received completion", completion)
    }
}
```
1. An optional `subscription` property is defined, which will be used to store the subscription
1. The subscriber requests up to a max of `2` values when it receives the subscription, and it stores the subscription

In `receive(_:)` this subscriber implementation is dynamically adjusting its demand. The returned demand will be the new demand _once the previous demand is fully satisfied_, so...

3. The new `max` is `3` (original `max` of `2` + new `max` of `1`).
1. `max` will remain `3` (previous `3` + `none`)

Continuing the example:

```swift
// 5
let subscriber = IntSubscriber()

// 6
let subject = PassthroughSubject<Int, Never>()

subject.subscribe(subscriber)

// 7
subject.send(1)
subject.send(2)
subject.send(3)
subject.send(4)
subject.send(5)
```
5. Create an instance of `IntSubscriber`.
1. Create a passthrough subject and subscribe to it.
1. Sends some values on the subject.

This will print:

```none
Received value 1
Received value 2
Received value 3
```

The first value the subscriber receives is `1`, and it returns `.max(2)`, so the new max is `3` (original `max` of `2` + new `max` of `1`). Then for the remaining values it receives, it returns a demand of `.none`, so the the demand remains `3`.

In order to see all publishing events, you can use `print()` in the subscription:

```swift
subject
    .print()
    .subscribe(subscriber)
```

Now this will be printed out:

```none
receive subscription: (PassthroughSubject)
request max: (2)
receive value: (1)
Received value 1
request max: (1) (synchronous)
receive value: (2)
Received value 2
receive value: (3)
Received value 3
```

Notice the publisher has not completed, and the subscription has not been canceled. The subscription is _paused_.

I'll comment out the `print()` and add the following code to the example:

```swift
// 8
subscriber.subscription?.request(.max(2))

// 9
subject.send(6)
subject.send(7)
subject.send(8)
```
8. Call  `request(_:)` on the subscription, setting a new max of `2`.
1. Send some more values.

This will print:

```none
Received value 1
Received value 2
Received value 3
Received value 6
Received value 7
```

The subscription has been _resumed_, and the new max is fulfilled.

There is one caveat here though. If the publisher completes before the subscription is resumed, it will _not_ receive any more values or the completion event (i.e., the completion event is not _replayed_).

I'll adjust the example to demonstrate this by uncommenting the `print()` and then inserting a `send(completion: .finished)` before the `request(_:)`:

```swift
subject.send(completion: .finished)
print("--")
// 8
subscriber.subscription?.request(.max(2))
```

This will print:

```none
receive subscription: (PassthroughSubject)
request max: (2)
receive value: (1)
Received value 1
request max: (1) (synchronous)
receive value: (2)
Received value 2
receive value: (3)
Received value 3
receive finished
Received completion finished
--
request max: (2)
```

## Where can I learn more?

Check out this book that I am a co-author and the technical editor for:

[Combine: Asynchronous Programming with Swift](https://store.raywenderlich.com/a/742/link/27)

<a href="https://store.raywenderlich.com/a/742/link/27" alt="Combine: Asynchronous Programming with Swift"><img src="https://github.com/learncombine/Combine101/blob/master/images/combineBook.png"></a>

It's packed with coverage of Combine concepts, hands-on exercises in playgrounds, and complete iOS app projects. Everything you need to _transform yourself_ from novice to expert with Combine — and have fun while doing it!

Combine and SwiftUI are Copyright © 2019 Apple Inc.

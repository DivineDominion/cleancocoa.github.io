---
title: "SwiftyBeaver: Record to Array"
created_at: 2017-04-25 07:42:30 +0200
tags: [ log, error ]
comments: on
---

I am using [SwiftyBeaver](https://github.com/SwiftyBeaver/SwiftyBeaver) in TableFlip and my latest project. Doing some robust console and file logging is important, and this library seems to work just fine for my needs.

Except for error reporting. I do not want to attach the whole debug log when folks may report a simple problem. So I figured: maybe it'll help to have the last 5 or 10 log messages attached upon a first error encounter.

So I wrote a SwiftyBeaver "in memory" destination that is set up next to a file and a console destination:


```swift
class InMemoryDestination: BaseDestination {

    var maxHistory = 5
    fileprivate(set) var messages: [String] = []

    override func send(_ level: SwiftyBeaver.Level, msg: String, thread: String, file: String, function: String, line: Int) -> String? {

        let formattedString = super.send(level, msg: msg, thread: thread, file: file, function: function, line: line) ?? "\(msg) (formatting error!)"

        messages = Array(messages
            .appending(formattedString)
            .suffix(maxHistory))

        return formattedString
    }
}
```

As a means to access these messages, I use a free function-like global closure:

```swift
let error = ...
let logMessages = lastLogMessages().joined(separator: "\n")
reportError(error: error, additionalInfo: logMessages)
```

Here, `lastLogMessages` is a closure so it can be swapped during runtime:

```swift
fileprivate(set) var lastLogMessages: () -> [String] = { return [] }
```

This then comes in handy when I set up my SwiftyBeaver logger:

```swift
let console = ...
let file = ...
let inMemory: InMemoryDestination = {
    let inMemory = InMemoryDestination()
    inMemory.maxHistory = 10
    inMemory.format = "$DHH:mm:ss: $M"
    inMemory.asynchronously = false

    // Here create a closure that points inside the InMemoryDestination:
    lastLogMessages = { return inMemory.messages }

    return inMemory
}()

SwiftyBeaver.addDestination(console)
SwiftyBeaver.addDestination(file)
SwiftyBeaver.addDestination(inMemory)
```

And that's it! I get a buffer of 10 log messages and read access to said buffer when I prepare error report emails.

One step closer to releasing a version into the wild.

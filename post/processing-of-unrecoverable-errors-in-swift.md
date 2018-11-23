---
title: "Processing of Unrecoverable Errors in Swift"
date: 2018-11-16T16:22:03+03:00
draft: true
description: "There are unrecoverable errors. We do special runtime checks to anticipate those errors and if that check fails we call fatalError(_:file:line), assertionFailure(_:file:line:) or other similar terminating function. In what circumstances what exact function should we call?"
---

## Unrecoverable Errors

All events which programmers call "errors" can be separated to two types.

* Events caused by external factors. Like network connection failure.
* Events caused by programmer's mistake. Like reaching a switch operator case which should be unreachable. 

We process events of the first type in regular control flow. For example we react to network failure by showing message to a user and setting app for waiting of network connection recovery.

We try to find out and exclude events of the second type as early as possible, before code goes to production. One of the approach here is to run some [runtime checks terminating program execution](https://developer.apple.com/documentation/swift/swift_standard_library/debugging_and_reflection) in a debuggable sate and print message with indication of where in code the error is happened.

For example programmer may terminate execution if required initializer wasn't provided but was called. That will be inevitably noticed and fixed at first test run.

```swift
required init?(coder aDecoder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
}
```

Another example is the switching between indices (let's assume that for some reason you can't use enumeration).

```swift
switch index {
case 0:
    // something is done here
case 1:
    // other thing is done here
case 2:
    // and other thing is done here
default:
    assertionFailure("Impossible index")
}
```

Again programmer is going to get crash during debugging here in oder to inevitably notice bug in indexing.

There are five terminating functions from the Swift Standard Library (as for Swift 4.2).

```swift
func precondition(_ condition: @autoclosure () -> Bool, _ message: @autoclosure () -> String = default, file: StaticString = #file, line: UInt = #line)
```
```swift
func preconditionFailure(_ message: @autoclosure () -> String = default, file: StaticString = #file, line: UInt = #line) -> Never
```
```swift
func assert(_ condition: @autoclosure () -> Bool, _ message: @autoclosure () -> String = default, file: StaticString = #file, line: UInt = #line)
```
```swift
func assertionFailure(_ message: @autoclosure () -> String = default, file: StaticString = #file, line: UInt = #line)
```
```swift
func fatalError(_ message: @autoclosure () -> String = default, file: StaticString = #file, line: UInt = #line) -> Never
```

**Which of five terminating functions should we prefer?**

## Source Code vs Documentation

Let's look at [source code](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift). We can see right away the following.

1. Every one of these five functions either terminates program execution or just doing nothing.
2. Possible termination happens by two ways.
	* Either with printing convenient debug message by calling `_assertionFailure(_:_:file:line:flags:)`.
	* Or without debug message just by calling `Builtin.condfail(error._value)` or `Builtin.int_trap()`.
3. The difference between five terminating functions is in the conditions when all of the above happens.
4. `fatalError(_:file:line)` calls `_assertionFailure(_:_:file:line:flags:)` unconditionally.
4. The other four terminating functions evaluate conditions by calling the following configuration evaluation functions. (They begin with underscore which means that they are internal and are not supposed to be called directly by programmer who uses Swift Standard Library).
	* `_isReleaseAssertConfiguration()`
	* `_isDebugAssertConfiguration()`
	* `_isFastAssertConfiguration()`

Then let's look at [documentation](https://developer.apple.com/documentation/swift/swift_standard_library/debugging_and_reflection). We can see right away the following.

1. `fatalError(_:file:line)` [unconditionally prints a given message and stops execution](https://developer.apple.com/documentation/swift/1538698-fatalerror).
2. The other four terminating functions' "effects vary depending on the build flag used": `-Onone`, `-O`, `-Ounchecked`. For example look at `preconditionFailure(_:file:line:)` [documentation](https://developer.apple.com/documentation/swift/1539374-preconditionfailure).
3. We can set these build flags in Xcode through `SWIFT_OPTIMIZATION_LEVEL` compiler build setting.
4. Also from Xcode 10 [documentation](https://help.apple.com/xcode/mac/10.0/#/itcaec37c2a6) we know that one more optimization flag — `-Osize` — is introduced.
5. So we have the four optimization build flags to consider.
	* `-Onone` (don't optimize)
	* `-O` (optimize for speed)
	* `-Osize` (optimize for size)
	* `-Ounchecked` (switch off many compiler checks)

We may conclude that configuration evaluated in four terminating functions is set by those build flags.

## Running Configuration Evaluation Functions

Although configuration evaluation functions are for internal usage some of them are [public for testing purposes](https://github.com/apple/swift/commit/d0697f2ac1092f74548c4df348194a3ee9ea7cda) so we may try them through CLI giving the following commands in Bash.

```bash
$ echo 'print(_isFastAssertConfiguration())' >conf.swift
$ swift conf.swift
false
$ swift -Onone conf.swift
false
$ swift -O conf.swift
false
$ swift -Osize conf.swift
false
$ swift -Ounchecked conf.swift
true
```

```bash
$ echo 'print(_isDebugAssertConfiguration())' >conf.swift
$ swift conf.swift
true
$ swift -Onone conf.swift
true
$ swift -O conf.swift
false
$ swift -Osize conf.swift
false
$ swift -Ounchecked conf.swift
false
```

Those tests and [source code](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift) inspection lead us to the following not very strict conclusions.

There are three mutually exclusive configurations.
	
* *Release* configuration is set by providing either `-O` or `-Osize` build flag.
* *Debug* configuration is set by providing either `-Onone` build flag or none optimization flags at all.
* `_isFastAssertConfiguration()` is evaluated to `true` if `-Ounchecked` build flag is set. And although this function has a word "fast" in it's name it has nothing to do with optimizing for speed `-O` build flag in spite of the misleading naming.

**NB:** Those conclusions are not strict definition of when *debug* builds or *release* builds are taking place. It's more complex issue. But those conclusions are correct for the context of terminating functions usage.

## Simplifyig The Picture

### `-Ounchecked`

Let's look not at **what is `-Ounchecked` flag for** (it's irrelevant here) but at **what is its role** in the context of terminating functions usage.

* Documentation for `precondition(_:_:file:line:)` and `assert(_:_:file:line:)` says: "In `-Ounchecked` builds, condition is not evaluated, but the optimizer may assume that it always evaluates to true. Failure to satisfy that assumption is a serious programming error."
* Documentation for `preconditionFailure(_:file:line)` and `assertionFailure(_:file:line:)` says: "In `-Ounchecked` builds, the optimizer may assume that this function is never called. Failure to satisfy that assumption is a serious programming error."
* We can see from [source code](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift) that evaluation of `_isFastAssertConfiguration()` to `true` **shouldn't happen**. (If it does happen peculiar `_conditionallyUnreachable()` is called, see [136 line and 176 lines](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift).

Speaking more directly, you **must not allow reachability** of the following four terminating functions with `-Ounchecked` build flag set for your program.

* `precondition(_:_:file:line:)`
* `preconditionFailure(_:file:line)`
* `assert(_:_:file:line:)`
* `assertionFailure(_:file:line:)`

Use only `fatalError(_:file:line)` while applying `-Ounchecked` and at the same time considering that the point of your program with `fatalError(_:file:line)` instruction is reachable.

### Condition Check Role

Two of terminating functions let us check for conditions. [Source code](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift) inspection let us see that if condition is failed then function behavior is the same as behavior of its respective cousin:

* `precondition(_:_:file:line:)` becomes `preconditionFailure(_:file:line)`,
* `assert(_:_:file:line:)` becomes `assertionFailure(_:file:line:)`.

That knowledge simplify our further analysis.

###  Release vs Debug Configurations

Further documentation and source code inspection let us eventually formulate the following table.

![terminating functions](/fatal_functions.png)

It's clear now that the most important choice for a programmer is what should be a program behavior **in _release_** when runtime check reveals an error.

The key take away here is that `assert(_:_:file:line:)` and `assertionFailure(_:file:line:)` let program fail *not so badly*. For example iOS app may have corrupted UI (since some important runtime checks was failed) but wan't crash.

But that scenario may easy be not the one you wanted. You have a choice.

## `Never` Return Type

`Never` is used as a return type of functions that unconditionally throw an error, traps, or otherwise do not terminate normally. Those kind of funstions do not actually return, they **never** return.

Among five terminating functions only `preconditionFailure(_:file:line)` and `fatalError(_:file:line)` return `Never` because only those two functions unconditionally stop program execution therefore never return.

Here is a nice example of utilizing `Never` type in some command line app. (Although this example uses not SwiftStandard Library terminating functions but standard C `exit()` function).

```swift
func printUsagePromptAndExit() -> Never {
    print("Usage: command directory")
    exit(1)
}
guard CommandLine.argc == 2 else {
    printUsagePromptAndExit()
}

// ...
```

If `printUsagePromptAndExit()` returned `Void` instead of `Never` you would get a buildtime error with the message: "*'guard' body must not fall through, consider using a 'return' or 'throw' to exit the scope*". Using `Never` you are saying in advance that you **never** exit the scope and therefore compiler won't give you a buildtime error. Otherwise you should add `return` at the end of guard code block, which isn't look nice.

## Takeaways

* It doesn't matter what terminating function to use if you are sure that all your runtime checks are relevant only for *Debug* configuration.
* Use only `fatalError(_:file:line)` while applying `-Ounchecked` and at the same time considering that the point of your program with `fatalError(_:file:line)` instruction is reachable.
* Use `assert(_:_:file:line:)` and `assertionFailure(_:file:line:)` if you are afraid that runtime checks may fail somehow in release. At least your app wan't crash.
* Use `Never` to make your code look neat.

## Useful Links

* [WWDC video "What's New in Swift"](https://developer.apple.com/videos/play/wwdc2018/401) telling about `SWIFT_OPTIMIZATION_LEVEL` build setting (from 11 minute).
* [How Never Works Internally in Swift](https://swiftrocks.com/how-never-works-internally-in-swift.html)
* [NSHipster's article about nature of `Never`](https://nshipster.com/never/)
* [Swift Forums discussion](https://forums.swift.org/t/deprecating-ounchecked/6928) about suggestion to deprecate `-Ounchecked`. 
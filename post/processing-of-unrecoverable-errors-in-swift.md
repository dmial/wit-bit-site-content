---
title: "Processing of Unrecoverable Errors in Swift"
date: 2018-11-16T16:22:03+03:00
draft: true
description: "There are places of your code which shouldn't be reachable during normal execution process. If that place is reached you should stop program execution and signal about programmer's (probably yours) error. How? Call for fatalError(), assertionFailure() or what?"
---

## Issue Description

All events which we call "errors" can be separated to two types.

* Events caused by external factors. Like network connection failure.
* Events caused by programmer's mistake. Like reaching a switch operator case which should be unreachable. 

We process events of the first type in regular control flow. For example we react to network failure by showing message to a user and setting app for waiting of network connection recovery.

We are trying to find out events of the second type as early as possible, before code goes to production. One of the approach here is to terminate program execution in a debuggable sate and print message with indication of where in code the event happened.

And that we can make with the following five functions from the Swift Standard Library (as for Swift 4.2).

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

Let's call them *fatal functions* for simple identification for the rest of this article.

**Which fatal function should we prefer?**

## Source Code vs Documentation

Let's look at [source code](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift). We can see right away the following.

1. Every one of these five functions either terminates program execution or just doing nothing.
2. Possible termination happens by two ways.
	* Either with printing convenient debug message by calling `_assertionFailure(_:_:file:line:flags:)`.
	* Or without debug message just by calling `Builtin.condfail(error._value)` or `Builtin.int_trap()`.
3. The difference between our five functions is in the conditions when nothing is happened or when a termination happens (with a call to `_assertionFailure(_:_:file:line:flags:)` or without). **And that is the main question of the article answered at the end of it.**
4. `fatalError(_:file:line)` calls `_assertionFailure(_:_:file:line:flags:)` unconditionally.
4. The other four *fatal functions* evaluate conditions by calling the following *configuration evaluation functions*.
	* `_isReleaseAssertConfiguration()`
	* `_isDebugAssertConfiguration()`
	* `_isFastAssertConfiguration()`

Then let's look at [documentation](https://developer.apple.com/documentation/swift/swift_standard_library/debugging_and_reflection). We can see right away the following.

1. `fatalError(_:file:line)` [unconditionally prints a given message and stops execution](https://developer.apple.com/documentation/swift/1538698-fatalerror).
2. The other four *fatal functions*' "effects vary depending on the build flag used": `-Onone`, `-O`, `-Ounchecked`. For example look at `preconditionFailure(_:file:line:)` [documentation](https://developer.apple.com/documentation/swift/1539374-preconditionfailure).
3. We can set these build flags in Xcode through SWIFT_OPTIMIZATION_LEVEL compiler build setting.
4. Also from Xcode 10 [documentation](https://help.apple.com/xcode/mac/10.0/#/itcaec37c2a6) we know that one more optimization flag — `-Osize` — is introduced.
5. So we have the four following build flags to consider.
	* `-Onone` (compile without any optimization; it's equal to not providing flag at all)
	* `-O` (optimize for speed)
	* `-Osize` (optimize for size)
	* `-Ounchecked` (switch off many compiler checks)

We may conclude that configuration evaluated in four *fatal functions* is set by those build flags.

## Running *Configuration Evaluation Functions*

Two of *configuration evaluation functions* are [public](https://github.com/apple/swift/blob/cec6d693374b201ce85d54f92cd38484f99ec55d/stdlib/public/core/AssertCommon.swift) so we may try them through CLI giving the following commands in Bash.

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

* All configurations are considered mutually exclusive.
	* *Release* configuration is set by providing either `-O` or `-Osize` build flag.
	* *Debug* configuration is set by providing either `-Onone` build flag or none flags at all.
	* `_isFastAssertConfiguration()` is evaluated to `true` if `-Ounchecked` build flag is set. And this configration has nothing to do with optimizing for speed `-O` build flag in spite of the misleading naming. So let's not call this configuration fast and just say that `_isFastAssertConfiguration()` is evaluated to `true`.

**NB:** Those conclusions are not strict definition of when *debug builds* or *release builds* are taking place. It's more complex issue. But those conclusions are correct for the context of *fatal functions* usage.

## Simplifyig The Picture

### `-Ounchecked`

* Documentation for `precondition(_:_:file:line:)` and `assert(_:_:file:line:)` says: "In `-Ounchecked` builds, condition is not evaluated, but the optimizer may assume that it always evaluates to true. Failure to satisfy that assumption is a serious programming error."
* Documentation for `preconditionFailure(_:file:line)` and `assertionFailure(_:file:line:)` says: "In `-Ounchecked` builds, the optimizer may assume that this function is never called. Failure to satisfy that assumption is a serious programming error."
* We can see from [source code](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift) that evaluation of `_isFastAssertConfiguration()` to `true` **shouldn't happen**. See [136 line and 176 line](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift).

Speaking more directly, you **must not allow reachability** of the following four fatal functions with `-Ounchecked` build flag set for you program.

* `precondition(_:_:file:line:)`
* `preconditionFailure(_:file:line)`
* `assert(_:_:file:line:)`
* `assertionFailure(_:file:line:)`

Use only `fatalError(_:file:line)` if you apply `-Ounchecked` and at the same time consider that the point of your program with `fatalError(_:file:line)` instruction is reachable.

If you are interested, there are [discussions](https://forums.swift.org/t/deprecating-ounchecked/6928) about depricating `-Ounchecked`. 

### Condition Check Role

Two of *fatal functions* let us check for conditions. [Source code](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift) inspection let us see that if condition is failed then function behavior is the same as behavior of its respective cousin: `precondition(_:_:file:line:)` becomes `preconditionFailure(_:file:line)`, `assert(_:_:file:line:)` becomes `assertionFailure(_:file:line:)`.

###  Release vs Debug Configurations

So we narrow down our analysis to the three *fatal functions* — `preconditionFailure(_:file:line)`, `assertionFailure(_:file:line)`, `fatalError(_:file:line)` — and their behavior with `-O`, `-Osize` build flags or without them (which is equal to `-Onone` flag).

And inspecting documentation and source code again we can eventually answer the **the main question of the article**: when nothing is happened or when a termination happens (with a call to `_assertionFailure(_:_:file:line:flags:)` or without).

Here is the answer.

<table>

## *Never* Return Type

TODO: How may it be useful and how does it influence function selection.

## Useful Links

* [WWDC video What's New in Swift](https://developer.apple.com/videos/play/wwdc2018/401) telling about SWIFT_OPTIMIZATION_LEVEL build setting from 11 minute.
* https://swiftrocks.com/how-never-works-internally-in-swift.html
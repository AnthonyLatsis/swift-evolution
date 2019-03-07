# Key-Path Sorting

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Anthony Latsis](https://github.com/AnthonyLatsis)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)

## Introduction

This proposal presents a minor addition to the Standard Library in an effort to make closureless key-path sorting a functional prerequisite for Swift.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

The easiest way to sort a collection over a `Comparable` property of its `Element` type is to use a regular predicate.

```swift
struct Person {
    var age: UInt8
    ...
}

struct Chat {
    var lastMessage: Message
    ...
}

var people: [Person] = ...
var chats: [Chat] = ...

people.sort { $0.age < $1.age }
chats.sort { $0.lastMessage.date > $1.lastMessage.date }
```

With type-safe key-path expressions introduced in **Swift 4**, it couldn't have been easier and more transparent:
we can get rid of the closure and therefore retain the argument label, giving the call a surprising resemblance to actual human language.

```swift
people.sort(by: \.age)
chats.sort(by: \.lastMessage.date, ascending: false)
``` 

The API, however, is not out-of-the-box; you will have to write the convenience youself.
As shown in the implementation section below, making this work requires just a bit of indirection anyone could figure out.
On the other hand, just like [`sort()`](https://developer.apple.com/documentation/swift/mutablecollection/2802575-sort)
and [`sorted()`](https://developer.apple.com/documentation/swift/sequence/1641066-sorted), key-path sorting a is highly
common practice and a fundamental sorting case that deserves a convenience method in the Standard Library, the author believes.
Since key-path expressions are relatively new to Swift, these additions also bear the advantage of promoting usage
of a modern language feature.

## Proposed solution

Add an overload for both the non-mutating `sorted` and in-place `sort` methods on `Sequence` and `MutableCollection` respectively.

```swift
extension Sequence {
    
    @inlinable
    public func sorted<Value: Comparable>(
        by metric: KeyPath<Element, Value>,
        ascending: Bool = true
    ) -> [Element] {
        if ascending {
            return sorted {
                $0[keyPath: metric] < $1[keyPath: metric]
            }
        }
        return sorted {
            $0[keyPath: metric] > $1[keyPath: metric]
        }
    }
}

extension MutableCollection where Self: RandomAccessCollection {
    
    @inlinable
    public mutating func sort<Value: Comparable>(
        by metric: KeyPath<Element, Value>,
        ascending: Bool = true
    ) {
        if ascending {
            sort {
                $0[keyPath: metric] < $1[keyPath: metric]
            }
        } else {
            sort {
                $0[keyPath: metric] > $1[keyPath: metric]
            }
        }
    }
}
```

## Source compatibility & ABI stability

This is an ABI-compatible addition with no impact on source compatibility.


## Alternatives considered

The implementation could be generalized to accept arbitrary predicates of type `(Value, Value) -> Bool`.
The demand on custom predicates in this area is less clear, but we get the appealing bonus of directly passing operator functions: 
`chats.sort(by: \.lastMessage.date, >)`

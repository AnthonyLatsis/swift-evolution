# Ban generic parameter direct shadowing in type & extension declaration contexts

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Anthony Latsis](https://github.com/AnthonyLatsis)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

This proposal advises disallowing a special case of generic parameter shadowing in type and extension declaration contexts: a nested type declaration that shadows a generic parameter of its immediately enclosing type or extension declaration will become a redeclaration error.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

Shadowing rules in Swift are able to be rather loose due to its well designed grammar and type system. Almost any nominal declaration can be shadowed, and this also holds true when shadowing has no practical value.

``` swift
let number: Int? = 0

if let number = number {
    let number = "0"
}
```

In a sense, the special case this proposal focuses on stands before the roadmap of the language, but it can also hardly be called a reasonable shadowing rule in Swift: the generic parameter and the shadowing type share the same declaration context.

Today, generic parameters can be shadowed with type aliases or nested type declarations. For the current declaration context, this means occurences of the relevant identifier will resolve to the new type rather than the generic parameter. In function and local contexts only subsequent occurences are affected:

```swift
func foo<U>(arg: U) {
    let x = arg // OK
    typealias U = Int
    let y = arg // Cannot convert value of type 'U' to specified type 'Int'
}
```

However, things are different in the body of a nominal type or extension declaration, where forward referencing *is* allowed. Not only all ocurrences are affected, but the shadowing also happens and can be realized retroactively.

```swift
struct Box<T> {
    let value: T
    typealias T = Int
}

let box = Box<String>(value: "Hello World") // Cannot convert value of type 'String' to expected argument type 'Int'
```

#### Diagnostics

Diagnostic messages are intended to help a user track down the source of the error. Allowing for shadowing of type identifiers within a single declaration context in a model to which that is alien results in misleading and confusing errors.

```swift
extension Box {
    func copy() -> Box<T> {
        return Box(value: value) // Cannot convert return expression of type 'Box<T>' to return type 'Box<Box<T>.T>'
    }
}
```

Furthermore, shadowing a generic parameter *unintentionally* through an extension might not affect other code and remain unnoticed until some of that code is put to use. This can happen if a developer declares a nested type or a type alias that turns out to match a generic parameter. Where the majority would rely on a redeclaration error, even an error is not guaranteed. In fact, a type lookup ambiguity can only occur when extending types from a precompiled library. Granted, this scenario is fairly rare, but must be considered as part of a bigger picture.

#### Retroactive implications

Shadowing a generic parameter retroactively can break code. This is clearly a breach in the established resilience model that strives to keep clear of mechanisms capable of retroactively breaking source. We cannot extend enums with new cases or protocols with requirements for that very reason. Occasions with shadowing sometimes force the compiler to magically ignore technically correct albeit harmful code when the target declaration source is inaccessible to the consumer, i.e. an SDK framework.

```swift
struct Box<T> {
    let value: T

    func toArray<U>() -> [Box<U>] where T: Sequence, T.Element == U {
        return value.map { Box<U>(value: $0) }
    }
}

extension Box {
    typealias T = Int 
    // Breaks 'toArray' and any other previously declared API of 
    // 'Box' that doesn't expect 'T == Int'.
}
```
_ _ _ _ _

When shadowing occurs directly in the context of the generic type, it is exceedingly hard to come up with a use case that could serve a compelling argument in favor of having such a possibility. But the main reason to consider this proposal isn't just a matter of practicality or correctness. The concrete shadowing rules described above are also a hindrance to the future of the language. There are a number of wanted features that will affect this issue if implemented:

* **Accessing generic parameters through qualified lookup.** With the ability to access generic parameters using dot notation, cases discussed will be subject to conflicting accesses and shadowing effectively becomes a violation.
* **Allowing static properties on generic types.** Static properties also give room to analogous conflicts that naturally produce redeclaration errors elsewhere.
  ``` swift
  class Class {
      class Foo {}
      static let Foo = 0 // Invalid redeclaration of 'Foo'
  }
  ```

## Proposed solution

Change the shadowing rule so that it becomes a redeclaration error. Although generic parameters currently cannot be accessed from outside its type, they are technically part of the type's declaration context. 

## Source compatibility

Code that relies on such shadowing occasions will no longer work. However, because there is next to no reason to use this possibility, the impact is expected to be minor.

## Effect on ABI stability

None.

## Effect on API resilience

None, either than eliminating a non-resilient shadowing rule.

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

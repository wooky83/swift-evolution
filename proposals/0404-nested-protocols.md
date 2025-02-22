# Allow Protocols to be Nested in Non-Generic Contexts

* Proposal: [SE-0404](0404-nested-protocols.md)
* Authors: [Karl Wagner](https://github.com/karwa)
* Review Manager: [Holly Borla](https://github.com/hborla)
* Status: **Accepted**
* Implementation: [apple/swift#66247](https://github.com/apple/swift/pull/66247) (gated behind flag `-enable-experimental-feature NestedProtocols`)
* Review: ([pitch](https://forums.swift.org/t/pitch-allow-protocols-to-be-nested-in-non-generic-contexts/65285)), ([review](https://forums.swift.org/t/se-0404-allow-protocols-to-be-nested-in-non-generic-contexts/66332)), ([acceptance](https://forums.swift.org/t/accepted-se-0404-allow-protocols-to-be-nested-in-non-generic-contexts/66668))

## Introduction

Allows protocols to be nested in non-generic `struct/class/enum/actor`s, and functions.

## Motivation

Nesting nominal types inside other nominal types allows developers to express a natural scope for the inner type -- for example, `String.UTF8View` is `struct UTF8View` nested within `struct String`, and its name clearly communicates its purpose as an interface to the UTF-8 code-units of a String value.

However, nesting is currently restricted to struct/class/enum/actors within other struct/class/enum/actors; protocols cannot be nested at all, and so must always be top-level types within a module. This is unfortunate, and we should relax this restriction so that developers can express protocols which are naturally scoped to some outer type.

## Proposed solution

We would allow nesting protocols within non-generic struct/class/enum/actors, and also within functions that do not belong to a generic context.

For example, `TableView.Delegate` is naturally a delegate protocol pertaining to table-views. Developers should be declare it as such - nested within their `TableView` class:

```swift
class TableView {
  protocol Delegate: AnyObject {
    func tableView(_: TableView, didSelectRowAtIndex: Int)
  }
}

class DelegateConformer: TableView.Delegate {
  func tableView(_: TableView, didSelectRowAtIndex: Int) {
    // ...
  }
}
```

Currently, developers resort to giving things compound names, such as `TableViewDelegate`, to express the same natural scoping that could otherwise be expressed via nesting.

As an additional benefit, within the context of `TableView`, the nested protocol `Delegate` can be referred to by a shorter name (as is the case with all other nested types):

```swift
class TableView {
  weak var delegate: Delegate?
  
  protocol Delegate { /* ... */ }
}
```

Protocols can also be nested within non-generic functions and closures. Admittedly, this is of somewhat limited utility, as all conformances to such protocols must also be within the same function. However, there is also no reason to artificially limit the complexity of the models which developers create within a function. Some codebases (of note, the Swift compiler itself) make use of large closures with nested types, and they beneift from abstractions using protocols.

```swift
func doSomething() {

   protocol Abstraction {
     associatedtype ResultType
     func requirement() -> ResultType
   }
   struct SomeConformance: Abstraction {
     func requirement() -> Int { ... }
   }
   struct AnotherConformance: Abstraction {
     func requirement() -> String { ... }
   }
   
   func impl<T: Abstraction>(_ input: T) -> T.ResultType {
     // ...
   }
   
   let _: Int = impl(SomeConformance())
   let _: String = impl(AnotherConformance())
}
```

## Detailed design

Protocols may be nested anywhere that a struct/class/enum/actor may be nested, with the exception of generic contexts. For example, the following remains forbidden:

```swift
class TableView<Element> {

  protocol Delegate {  // Error: protocol 'Delegate' cannot be nested within a generic context.
    func didSelect(_: Element)
  }
}
```

The same applies to generic functions:

```swift
func genericFunc<T>(_: T) {
  protocol Abstraction {  // Error: protocol 'Abstraction' cannot be nested within a generic context.
  }
}
```

And to other functions within generic contexts:

```swift
class TableView<Element> {
  func doSomething() {
    protocol MyProtocol {  // Error: protocol 'Abstraction' cannot be nested within a generic context.
    }
  }
}
```

Supporting this would require either:

- Introducing generic protocols, or
- Mapping generic parameters to associated types.

Neither is in in-scope for this proposal, but this author feels there is enough benefit here even without supporting generic contexts. Either of these would certainly make for interesting future directions.

### Associated Type matching

When nested in a concrete type, protocols do not witness associated type requirements.

```swift
protocol Widget {
  associatedtype Delegate
}

struct TableWidget: Widget {
  // Does NOT witness Widget.Delegate
  protocol Delegate { ... }
}
```

Associated types associate one concrete type with one conforming type. Protocols are constraint types which many concrete types may conform to, so there is no obvious meaning to having a protocol witness an associated type requirement.

There have been discussions in the past about whether protocols could gain an "associated protocol" feature, which would allow these networks of constraint types to be expressed. If such a feature were ever introduced, it may be reasonable for associated protocol requirements to be witnessed by nested protocols, the same way associated type requirements can be witnessed by nested concrete types today.

## Source compatibility

This feature is additive.

## ABI compatibility

This proposal is purely an extension of the language's ABI and does not change any existing features.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints.

In general, moving a protocol in/out of a parent context is a source-breaking change. However, this breakage can mitigated by providing a `typealias` to the new name.

As with other nested types, the parent context forms part of the mangled name of a nested protocol. Therefore, moving a protocol in/out of a parent context is an ABI-incompatible change.

## Future directions

- Allow nesting other (non-protocol) types in protocols

  Protocols themselves sometimes wish to define types which are naturally scoped to that protocol. For example, the standard library's [`FloatingPointRoundingRule`](https://developer.apple.com/documentation/swift/FloatingPointRoundingRule) enum is used by a [requirement of the `FloatingPoint` protocol](https://developer.apple.com/documentation/swift/floatingpoint/round(_:)) and defined for that purpose.
  
- Allow nesting protocols in generic types

  As mentioned in the Detailed Design section, there are potentially strategies that would allow nesting protocols within generic types, and one could certainly imagine ways to use that expressive capability. The community is invited to discuss potential approaches in a separate topic.

## Alternatives considered

None. This is a straightforward extension of the language's existing nesting functionality.

## Acknowledgments

Thanks to [`@jumhyn`](https://forums.swift.org/u/jumhyn/) for reminding me about this, and to [`@suyashsrijan`](https://forums.swift.org/u/suyashsrijan/). 

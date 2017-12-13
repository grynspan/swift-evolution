# Cross-module inlining and specialization

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Slava Pestov](https://github.com/slavapestov)
* Review Manager: TBD
* Status: **Initial pitch**
* Implementation: Already implemented as a pair of underscored attributes `@_inlineable` and `@_versioned`

## Introduction

We propose introducing a pair of new attributes, `@inlinable` and `@abiPublic`. The `@inlinable` attribute exports the body of a function as part of a module's interface, making it available to the optimizer when referenced from other modules. The `@abiPublic` attribute which marks an internal declaration as being part of the binary interface of a module, allowing it to be used from `@inlinable` code without exposing it as part of the module's source interface.

## Motivation

One of the top priorities of the Swift 5 release is a design and implementation of _the Swift ABI_. This effort consists of three major tasks:

* Finalizing the low-level function calling convention, layout of data types, and various runtime data structures. The goal here is to maintain compatibility across compiler versions, ensuring that we can continue to make improvements to the Swift compiler without breaking binaries built with an older version of the compiler.

* Implementing support for _library evolution_, or the ability to make certain source-compatible changes, without breaking binary compatibility. Examples of source-compatible changes we are considering include adding new stored properties to structs and classes, removing private stored properties from structs and classes, adding new public methods to a class, or adding new protocol requirements that have a default implementation. The goal here is to maintain compatibility across framework versions, ensuring that framework authors can evolve their API without breaking binaries built against an older version of the framework. For more information about the resilience model, see the
[library evolution
document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst)
in the Swift repository.

* Stabilizing the API of the standard library. The goal here is to ensure that the standard library can be deployed separately from client binaries and frameworks, without forcing recompilation of existing code.

All existing language features of Swift were designed with these goals in mind. In particular, the implementation of generic types and functions relies on runtime reified types to allow separate compilation and type checking of generic code.

Within the scope of a single module, the Swift compiler performs very aggressive optimization, including full and partial specialization of generic functions, inlining, and various forms of interprocedural analysis.

On the other hand, across module boundaries, runtime generics introduce unavoidable overhead, as reified type metadata must be passed between functions, and various indirect access patterns must be used to manipulate values of generic type. We believe that for most applications, this overhead is negligible compared to the actual work performed by the code itself.

However, for some advanced use cases, and in particular for the standard library, the overhead of runtime generics can dominate any useful work performed by the library. Examples include the various algorithms defined in protocol extensions of `Sequence` and `Collection`, for instance the `map` method of the `Sequence` protocol. Here the algorithm is very simple and spends most of its time manipulating generic values and calling to a user-supplied closure; specialization and inlining can completely eliminate the algorithm of the higher-order function call and generate equivalent code to a hand-written loop manipulating concrete types.

We would like to annotate such functions with the `@inlinable` attribute. This will make their bodies available to the optimizer when building client code. The optimizer may or may not make use of the function body; it might be inlined, specialized, or ignored, in which case the compiler will continue to reference the public entry point in the framework. If the framework were to change the definition of such a function, only binaries built against the newer version of library might continue using the old, inlined definition, they may use the new definition, or even a mix depending if certain call sites inlined the function or not.

## Proposed solution

The `@inlinable` attribute causes the body of a function to be emitted as part of the module interface. For example, a framework can define a rather impractical implementation of an algorithm which returns `true` if all elements of a sequence are equal or if the sequence is empty, and `false` otherwise:

```swift
@inlinable public func allEqual<T>(_ seq: T) -> Bool
    where T : Sequence, T.Element : Equatable {
  var iter = seq.makeIterator()
  guard let first = iter.next() else { return true }

  func rec(_ iter: inout T.Iterator) -> Bool {
    guard let next = iter.next() else { return true }
    return next == first && rec(&iter)
  }

  return rec(&iter)
}
```

A client binary built against this framework can call `allEqual()` and enjoy a possible performance improvement when built with optimizations enabled, due to the elimination of abstraction overhead.

On the other hand, once the framework author comes to their senses and implements an iterative solution to replace the recursive algorithm defined above, the client binary might not be able to make use of the more efficient implementation until recompiled.

## Detailed design

### The `@abiPublic` attribute

The `@abiPublic` attribute can be applied to all declarations which support access control modifiers. This includes almost all kinds of declarations, except for the following, which always have the same effective visibility as their containing declaration:

* Protocol requirements
* Enum cases
* Class destructors

When applied to a subscript or computed property, the attribute applies to all of the getter, setter, `didSet` and `willSet`, if present.

The `@abiPublic` attribute can be applied to `internal` declarations, and not `private`, `fileprivate` or `public` declarations. The `@abiPublic` attribute does not affect source-level visibility of a declaration; it only results in the entry point being exported at the ABI level, allowing it to be referenced from `@inlinable` functions.

This attribute allows us to introduce the notion of _ABI visibility_. A declaration is _ABI-public_ if the following conditions hold:

- The declaration is a top-level declaration, or it is nested inside an ABI-public type.
- The declaration must itself be `public`, or it can be `internal` and annotated with the `@abiPublic` attribute.

In the following example, the method `C.f` is ABI-public:

```swift
public class C {
  public func f() {}
}
```

Another example of an ABI-public declaration is the method `C.D.f` below:

```swift
public class C {
  @abiPublic internal class D {
    @abiPublic internal func f() {}
  }
}
```

In the following, the method `C.f` is **not** ABI-public, because it is nested inside a type that is not `@abiPublic` or `public`:

```swift
internal class C {
  public func f() {}
}
```

### The `@inlinable` attribute

The `@inlinable` attribute can be applied to the following kinds of declarations:

* Functions and methods
* Subscripts
* Computed properties
* Initializers
* Deinitializers

The attribute can only be applied to ABI-public declarations.

The attribute cannot be applied to local declarations, that is, declarations nested inside functions or statements. However, local functions  and closure expressions defined inside public `@inlinable` functions are always implicitly `@inlinable`.

When applied to a subscript or computed property, the attribute applies to all of the getter, setter, `didSet` and `willSet`, if present.

The compiler will enforce certain restrictions on bodies of inlinable declarations:

* inlinable declarations cannot define local types. This is because all types have a unique identity in the Swift runtime, visible to the language in the form of the `==` operator on metatype values. It is not clear what it would mean if two different libraries inline the same local type from a third library, with all three libraries linked together into the same binary. This becomes even worse if two _different_ versions of the same inlinable function appear inside the same binary.

* inlinable declarations can only reference ABI-public declarations. This is because they can be emitted into the client binary, and are therefore limited to referencing symbols that the client binary can reference.

**Note:** The restrictions enforced on the bodies of `@inlinable` declarations are exactly those that we have in place on default argument expressions of `public` functions in Swift 4. In particular, default argument expressions of public functions can now reference internal declarations, as long as those declarations are annotated with `@abiPublic`.

## Source compatibility

The introduction of the `@inlinable` and `@abiPublic` attributes is an additive change to the language and has no impact on source compatibility.

## Effect on ABI stability

Adding or removing `@inlinable` on an existing declaration does not change the ABI of that declaration.

Adding `@abiPublic` to an existing declaration does not change the ABI of that declaration. Removing `@abiPublic` does however, and therefore is a binary-incompatible change.

## Effect on API resilience

Any changes to the body of a declaration annotated as `@inlinable` should be considered very carefully. As a general guideline, we feel that `@inlinable` makes the most sense with "obviously correct" algorithms which manipulate other data types abstractly through protocols, so that any future changes to an `@inlinable` declaration are optimizations that do not change observed behavior.

Also, an `@inlinable` function implementation must be prepared to interact with multiple versions of the same function linked into a single binary. For example, if a hashing function is `@inlinable`, the hash algorithm must not be changed to avoid introducing inconsistency.

## Comparison with other languages

The closest language feature to the `@inlinable` attribute is found in C and C++. In C and C++, the concept of a header file is similar to Swift's binary `swiftmodule` files, except they are written by hand and not generated by the compiler. Swift's `public` declarations are roughly analogous to declarations whose prototypes appear in a header file.

Header files mostly contain declarations without bodies, but can also declare `static inline` functions with bodies. Such functions are not part of the binary interface of the library, and are instead emitted into client code when referenced. As with `@inlinable` declarations, `static inline` functions can only reference other "public" declarations, that is, those that are defined in other header files.

There is no equivalent to `@abiPublic` that we are aware of in other languages.

## Alternatives considered

One possible alterative would be to add a new compiler mode where _all_ declarations become implicitly `@inlinable`, and _all_ private and internal declarations become `@abiPublic`.

However, such a compilation mode would not solve the problem of delivering a stable ABI and standard library which can be deployed separately from user code. We _don't want_ all declaration bodies in the standard library to be available to the optimizer when building user code.

While such a feature might be useful for users who build private frameworks that are always shipped together their application without resilience concerns, we do not feel it aligns with our goals for ABI stability, and at best it should be a separate discussion.

For similar reasons, we do not feel that an "opt-out" attribute that can be applied to declarations to mark them non-inlinable makes sense.

We have also considered generalizing `@inlinable` to allow it to be applied to entire blocks of declarations, for example at the level of an extension. As we gain more experience with using this attribute in the standard library we might decide this would be a useful addition, but we feel that for now, it makes sense to focus on the case of a single inlinable declaration instead. Any future generalizations can be introduced as additive language features.

We originally used the spelling `@inlineable` for the attribute. However, we settled on `@inlinable` for consistency with the `Decodable` and `Encodable` protocols, which are named as they are and not `Decodeable` and `Encodeable`.

Finally, we have considered some alternate spellings for this attribute. The name `@inlinable` is somewhat of a misnomer, because nothing about it actually forces the compiler to inline the declaration; it might simply generate a concrete specialization of it, or look at the body as part of an interprocedural analysis, or completely ignore the body. However, nothing seemed to read as well as `@inlinable`.
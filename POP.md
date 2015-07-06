# Intro

My favorite session at WWDC this year was definitely [Protocol-Oriented
Programming in Swift][WWDC_POP]. It gave a name to a programming paradigm I've
been practicing (unwittingly) for a little while now, and highlighted the fact
that Swift was designed from the ground up with this paradigm in mind. Perhaps
that's why it works so well. The talk also went over some new language features
that would make working this way even nicer.

I know I'm not the only one who enjoyed this talk for these same reasons. I've
talked with a lot of people about this talk since WWDC and I find that the reactions
boiled down into three basic types:

  1. I loved it! Favorite talk of the year! - These people cite mostly the same
  reasons as me for loving the talk so much.
  2. It blew my mind! I loved it! Favorite talk of the year! - These people were
  excited by the new ideas presented in the talk and were excited to bring the
  approach to their projects as soon as possible, but when they went Googling
  for some more info, they came up empty handed.
  3. That seemed interesting, but it was way over my head... I'm gonna come back
  to that one later.

What I wanted to do with this article is go over some things that I've picked up
while doing Protocol-Oriented Programming before I knew it had a name in the hopes
of giving those in camp 2 a little something to get started, some "best practices"
as it were.

I may do another article for folks in camp 3, but it will be of a different style
(I'm thinking timestamp annotated notes for the video, like a companion guide).

# Getting started

Protocol-Oriented Programming is definitely not an all or nothing approach. You
can start mixing it in to your project today, either by refactoring pieces of
your current components or by adding a new component designed in this way.
Protocols in Swift can be applied to and used by any type (be it `class`, `enum`,
or `struct`), and so long as the interface described by the protocol can be used
by objective-c, it can be used there too, so you are free to mix them in as
slowly or whole-heartedly as you like!

If you are going to be integrating with an existing system though, note that you
may end up having to alter or refactor some pieces of that system that you aren't
fully converting to POP (...yet). This is just the nature of the beast when it
comes to refactoring for dependency injection (don't worry, I'll get to that if
you don't know what "dependency injection" means).

# Start With a Protocol

I know, he said this in the talk, but it's worth repeating and was one of the
things that he said where I thought "Huh... yeah, why didn't I think of that?!".
I used to just start with a `class` (or `struct` or `enum` where it made sense),
defining the properties and methods I thought it'd need and stubbing them out,
maybe even implementing them. Later I'd create a `protocol` from that public
interface. Now, I start by thinking about that public interface first, and
creating a `protocol` of it right from the start. Then I go and start adding an
implementation for that protocol in a `class` (or `struct`).

## Same File or Multiple Files?

I'm usually a pretty big stickler for single type per file, but I'm not a big fan
of tons of tiny files that are a pain to navigate through in Xcode, so when the
protocol is small enough (< 5 things in it), I'll usually just leave it at the
top of the file that has the type which implements it. But sometimes I'll find
myself creating multiple protocols for a single type; one for its input related
methods and one for its output related methods, when that makes sense. Usually
when a type has methods that can be separated out that way it's because the type
is being used as a bridge between two layers of the app (e.g. between the UI and
the database or network) so when the object is used elsewhere there is only a
subset of its interface that is important to either side, at which point it makes
sense to have it subdivided in this way. And once that starts happening, it's
probably a good idea to start breaking the protocols and the type into separate
files, else it just gets too busy.

# Value Semantics vs. Reference Semantics

I highly recommend reading "[A Warm Welcome to Structs and Value Types][objc.io_ValueTypes]"
on [objc.io][objc.io_ValueTypes]. It is written by [Andy Matuschak][AndyMatuschak],
and does a far better job of explaining the ins and outs of Value vs Reference
semantics than I could.

For those who don't know the difference and are too lazy to go read that article,
here's the ever-so-basic gist:

* Value Types are generally types that _are_ something. These are always copied
when they are passed around, AKA: pass-by-value. You would never expect modifying
one value-type variable to have any effect on another one.
* Reference Types are more behavioral in nature, they are the types that generally
_do_ something. These are types that are passed around by reference, when you
assign a reference-type variable to another variable and modify one, both will
see the changes.

So the great thing about protocols and Protocol-Oriented Programming in Swift is
that they can be applied freely between the two types of types, and make it so
that you can decide whether to use a `struct` or a `class` at implementation
time.

## Value Types

Use value types when you're modeling a piece of data, a _value_ to be passed
around. Types like `Int`, `Double`, `Bool`, `CGPoint` are all classic value types.

Any time you create a value type, be sure to make it conform to `Equatable`. If
you find it difficult to answer the question of how two instances of this type
could be _equatable_, this is a big indicator that you should make that type a
`class` instead.

## Reference Types

Use reference types when you're modeling _behavior_ or something that is tied to
some external state. Types like `UIWindow`, `NSURLSession`, `MyDataStore` are all
prime candidates for reference types.

Reference types (aka classes or objects) should _add_ behavior to value types,
or use them as a means of communicating with other reference types. They are
_composed_ of value types, unless some shared ownership is absolutely necessary.

Again I'm going to link to Andy's [article][objc.io_ValueTypes], but specifically
to the "[Object of Objects][objc.io_ValueTypes_Objects]" section in which he
describes when and more importantly **how** you should use reference types. Here's
the important bits with some emphasis added by me:

> Objects maintain state, **defined by values**, but those values can be considered and manipulated independently of the object. The value layer doesn’t really have state; it just represents and transmutes data. That data may or may not have higher-level meaning as state, depending on the context in which the value’s used.

> Objects perform side effects like I/O and networking, but data, computations, and non-trivial decisions ultimately driving those side effects all exist at the value layer. **The objects are like the membrane, channeling those pure, predictable results into the impure realm of side effects.**

> **Objects can communicate with other objects, but they generally send values, not references**, unless they truly intend to create a persistent connection at the outer, imperative layer.

## Gotchas

Something you should be aware of when creating your value types: if you add
properties which are themselves a reference, that breaks the copy by value
semantics of the type, because when copying the type, its references are copied
as references, not values. This can lead to some really awkward situations:

```
// Swift 2.0 / Xcode 7 Beta 2

class FooClass {
    var value = ""
    init(value: String) {
        self.value = value
    }
}

struct FooStruct: CustomStringConvertible {
    let bar: FooClass
    init(bar: FooClass) {
        self.bar = bar
    }

    var description: String {
        return self.bar.value
    }
}
```

Here, we have a class with a `String` variable property `value` and struct with
a `let` property of that `class`'s type `bar`. So, there's no `mutating` or
anything and we've been told that anything created with `let` is immutable so
we'd expect `bar` to never change, right?

```
let myRef = FooClass(value: "Initial")
let myConstant = FooStruct(bar: myRef)
```

Here, we define an instance of our `class`, then create an instance of our
`struct`, initializing it with our instance of our `class`. Both of which we
declare with `let`, and those **never** change, right?

```
myConstant              // Initial
myRef.value = "MUTATED"
myConstant              // MUTATED
```

Wait... WHAT?! That's a value type, declared into a `let` and my code doesn't have
the `mutating` keyword in it **ANYWHERE**!

This demonstrates how value types in Swift handle reference-type properties. All
the `let` in `FooStruct` (and in the declaration of `myRef`) really says is that
the _reference_ to `bar` or `myRef` will never change. It can't guarantee that
the object that it references won't change on its own and because `value` is
declared `var` in `FooClass` it is mutable and can be changed no matter how the
variable that holds it was declared.

In this case it would be simple to change `bar` to just be of type `String` and
the problem goes away, but I wanted to point out the non-obvious pitfall of having
reference type properties on your value types. Moral of the story: **Compose your
value types with other value types if at all possible!** And if it's not possible,
be aware of this behavior!

I also highly recommend checking out the [Building Better Apps with Value
Types][WWDC_ValueTypes] and [Optimizing Swift Performance][WWDC_Performance] WWDC
sessions for a bunch of jewels in regards to how value and reference types work
under the hood.

# Dependency Management

# Extensions

# Generics

# Tests

# Code Examples

# Summary

---
[AndyMatuschak]: https://twitter.com/andy_matuschak
[objc.io_ValueTypes]: http://www.objc.io/issues/16-swift/swift-classes-vs-structs/
[objc.io_ValueTypes_Objects]: http://www.objc.io/issues/16-swift/swift-classes-vs-structs/#the-object-of-objects

[WWDC_POP]: https://developer.apple.com/videos/wwdc/2015/?id=408
[WWDC_ValueTypes]: https://developer.apple.com/videos/wwdc/2015/?id=414
[WWDC_Performance]: https://developer.apple.com/videos/wwdc/2015/?id=409

---
layout: post
title:  "[swift] Dynamic keyword"
categories: swift ios

---


`dynamic` is a declaration modifier that you can apply to either function or variable declarations. It can be used within a class only and it tells the run time to use dynamic dispatch over static dispatch.


dynamic vs static dispatch
--------------------------

As Swift allows a class to override methods and properties declared in its superclasses, this means that the program has to determine at run time which method or property is being referenced and then perform an indirect call or indirect access. This technique is called dynamic dispatch and is applicable to types that support inheritance - classes. Value types like structs and enums cannot be extended through inheritance so they use static dispatch only.


Dispatch types in Swift
-----------------------

Swift actually supports three types of dispatch, ordered here from faster to slower:

1. direct dispatch aka static dispatch
2. table dispatch aka virtual dispatch
3. message dispatch

There are [more thorough explanations](https://www.raizlabs.com/dev/2016/12/swift-method-dispatch/) of how each of them work, but here is a quick summary.

With a **direct dispatch**, there is a direct reference to the memory location where the method code is stored and that reference never changes in the run time. In the following example:

```swift
class Point {
  func draw() {
    // draw implementation
  }
}
//...
let p = Point()
p.draw()
```

`Point` is a class that doesn't inherit any other class nor there is a class that inherits from it. Compiler knows that there is only one implementation of the `draw()` method. Whenever `draw()` is called on a `Point` instance, compiler always references the same memory location where the code of that method is stored.

Now, let's say that we have a generic `Shape` class and two classes that represent specific implementations of it - `Line` and `Circle`:

```swift
class Shape {
    func draw() {
        // default draw implementation
    }
}

class Line: Shape {
    override func draw() {
        // line implementation
    }
}

class Circle: Shape {
    override func draw() {
        // circle implementation
    }
}

let shapes = fetchShapesFromAPI()
for shape in shapes {
    shape.draw()
}
```

In this example direct dispatch can't be used when `draw()` is called. The compiler doesn't know in advance which of the three `draw()` implementations to point to because shape may be of different type in each iteration. So we need to use one of the other two dispatch types.

With a **table dispatch**, there is a lookup table of method pointers for each class in the class hierarchy - `Shape`, `Line` and `Circle`. Each table contains pointers for all methods in that class, including those that a class inherits from the parent classes. At the point of a method call, the program goes to the lookup table for the given class and finds the method pointer in it. That happens in run time.

Now, let's add `redraw()` method to the `Shape` class, which has some imaginary clean-up code and then calls `draw()` again.

```swift
class Shape {
    func draw() {
        // line implementation
    }
    func redraw() {
        // clean up code
        self.draw()
    }
}
...
for shape in shapes {
    shape.redraw()
}
```

That method is inherited but not overridden in the `Line` and `Circle` classes. With table dispatch, the lookup table for each of the three classes, `Shape`, `Line` and `Circle`, will have a copy of the reference to the `redraw()` method from the `Shape` class.

But with a **message dispatch**, there are no lookup tables. When the `redraw()` method is invoked, the program starts from the given class and then iterates the class hierarchy to find which class has the specified method implemented. For example, if an instance of the `Line` is in the array, program will check if there is a `redraw()` method in the `Line` class, since it isn't, it will move up to the `Shape` class and find it there.

Now, before we discuss the dynamic dispatch, let's briefly mention the compiler optimizations first.

Compiler optimizations
----------------------

[Swift compiler](https://swift.org/compiler-stdlib/#compiler-architecture) translates code from a human friendly form into a machine friendly form and in the process it tries to optimize it to make it run faster. In the context of method dispatch, it will favor faster types over the slower. If the compiler can be sure at compile time that a particular method is going to be referenced, he might devirtualize it or even inline it.

**[Devirtualization](https://blogs.unity3d.com/2016/07/26/il2cpp-optimizations-devirtualization/)** is a common compiler optimization tactic which changes a virtual method call into a direct method call.

**[Inlining](http://www.compileroptimizations.com/category/function_inlining.htm)** is a compiler optimization tactic which replaces a direct method call with the method code inline.


Great, now tell me about dynamic dispatch!
------------------------------------------

Well, you will see both virtual dispatch and message dispatch referred to as dynamic dispatch. And although they are different mechanism, they both are dynamic. But besides them being different mechanisms with different strengths and weaknesses, there is one other important distinction. Table dispatch is a part of pure Swift, while message dispatch is only supported in the Cocoa environments. The [Objective-C runtime library](https://novemberfive.co/blog/objective-c-runtime/) is actually the one providing the message dispatch mechanism.

That finally brings us to the strength of the message dispatch. Since there is only one reference to one given method or property it can be safely and easily modified at run time. And Objective-C runtime provides tools to do it. These tools are the basis of cool dynamic features like KVC, KVO and UIAppearance.

Where does `dynamic` keyword come into play?
--------------------------------------------

Pure Swift classes don't normally use message dispatch and while classes derived from NSObject could use it, Swift compiler may optimize the way their properties and methods are referenced, which may break beforementioned dynamic features. If you want to use any of these dynamic features, you need to use `dynamic` modifier to enforce the message dispatch on that property or method. It can be used on both Swift and NSObject classes.

`dynamic` also implicitly adds the `objc` attribute and makes the declared property or method available in the Objective-C part of the program. To use the `dynamic` modifier, you must import Foundation, as this includes NSObject and the core of the Objective-C runtime.

Anything else?
--------------

`dynamic` can also be used on methods declared in class extensions to allow to override them.

Let's modify the previous example:

```Swift
class Shape {
    func draw() {
        // default draw implementation
    }
}

extension Shape {
    func redraw() {
        // clean up code
        self.draw()
    }
}

class Line: Shape {
    override func draw() {
        // line implementation
    }
    override func redraw() {
        //
    }
}
```

We have moved the `redraw()` method to the class extension. But now if we want to override it in the subclass, the compiler will show an error. We can work around that by adding `dynamic` keyword to the `redraw()` declaration in the extension.

```Swift
//...
extension Shape {
    **dynamic** func redraw() {
        // clean up code
        self.draw()
    }
}
//...
```

Why does that happen? Extension methods use static dispatch, so they can't be overridden. By adding `dynamic` to their declaration we force them to use message dispatch which allows overriding.



Computer Science 101
--------------------

Programmers write code in high level programming languages (e.g. Swift, Objective-C) that are easy to understand for people but overly complex for computers. So the high level code is translated into a much more simple machine code, through a process called compilation. That process consists of multiple steps and each step results with a lower level code.

When a program is executed, the program code is read from the memory and executed instruction by instruction. If a given instruction stands for a method call, it needs to be determined where the code for that function is located in the memory and read the code to be executed from that location. Depending on the method dispatch type, this process of finding the correct memory location can take more or less time, thus affecting the overall program performance.


References
----------
* https://krakendev.io/blog/hipster-swift
* https://www.raizlabs.com/dev/2016/12/swift-method-dispatch/
* https://cocoacasts.com/what-does-the-dynamic-keyword-mean-in-swift-3/
* https://blogs.unity3d.com/2016/07/26/il2cpp-optimizations-devirtualization/
* https://en.wikipedia.org/wiki/LLVM#Intermediate_representation
* https://www.quora.com/Swift-can-be-compiled-with-the-LLVM-which-generates-Objective-C-code-out-of-it-and-then-Objective-C-will-be-compiled-to-machine-code-or-how-does-it-work/answer/Andrea-Ferro
* https://www.infoq.com/articles/swift-objc-runtime-programming
* https://developer.apple.com/swift/blog/?id=27
* https://developer.apple.com/library/prerelease/content/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithObjective-CAPIs.html#//apple_ref/doc/uid/TP40014216-CH4-ID57
* https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151207/001707.html
* https://realm.io/news/real-world-swift-performance/
* http://cocoasamurai.blogspot.hr/2010/01/understanding-objective-c-runtime.html
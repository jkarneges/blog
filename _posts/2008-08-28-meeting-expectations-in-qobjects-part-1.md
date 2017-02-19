---
layout: post
title:  Meeting expectations in QObjects (Part 1)
date:   2008-08-28 09:30:00
---
I've written a few articles on how I believe QObjects should behave, and I've even attempted to codify these behaviors in the [Delta Object Rules][dor]. However, I haven't offered many solutions on how to go about making QObjects that follow these rules. This is in part because I didn't really have any good solutions. For the past few years I've experimented with some different approaches though, and I'll outline them here.

<!--more-->

## Basic Approaches

The most common approach you may have seen in my newer code is the use of member `QTimer` objects for single shots to defer execution. This is the main way to solve the DOR DS problem. `QTimer::singleShot()` and `QMetaObject::invokeMethod()` can do this as well, but the key is to use `QTimer` as a real object so the single shots can be canceled. The ability to cancel allows solving SR.

Of course there's also the old-school technique of emitting signals as the last part of a code path. This is often a solution to SS.

Another old-school technique is for an object to monitor itself with `QPointer`, but I no longer consider this to be a real solution. In my revised article on SS, I discuss how the problem goes beyond destructors. You can use `QPointer` if the only state change you're worried about is destruction, but I'd say that's the least of your worries.

Therefore, `QTimer` and "signals last" are in, and `QTimer::singleShot()`, `QMetaObject::invokeMethod()`, and `QPointer` are out. Let's talk about `QTimer` and "signals last".

## "Signals Last"

The idea here is that you try to emit your signals at the end of your object's code path. So if you have X things to do while code runs in your object, and one of them is a signal, then you arrange your code such that the signal emit is the last thing that happens.

Why is this useful? Well, whenever you emit a signal you're leaving your object vulnerable. The slot at the other end of the signal might run code that inspects or changes your object (maybe it runs some member methods on your object). Is your object prepared for this? Is it in the right state? If you emit your signals last, then the idea is that your object has already performed all of its other operations and is now in a predictable state. Additionally, there's no worry of the object being in an unpredictable state upon return, because there's no code to execute upon return!

The "signals last" approach becomes a challenge when you need to emit two signals, or when you need to add a new signal into the middle of some existing code.

## QTimers

Use `QTimer` to defer a signal (or to defer the code that leads to a signal). If a directly-invoked member method would want to emit a signal, then you can split the method into two parts:

1. A method called by the user which simply starts a `QTimer`.
2. A deferred method, invoked by `QTimer`, which performs whatever the member method was originally supposed to perform.

In other words, this:

```c++
void MyClass::doStuff()
{
    something();
    somethingElse();
    emit stuffDone();
}
```

becomes:

```c++
void MyClass::doStuff()
{
    doStuffTimer.start(); // singleshot timeout calls doStuffDeferred
}

void MyClass::doStuffDeferred()
{
    something();
    somethingElse();
    emit stuffDone();
}
```

Alternatively you could perform all but the signal in the first part, and let the second part be just the signal emit. There is a difference when you do this, and I'll discuss what that means later on.

Unfortunately, the `QTimer` approach gets unwieldy for complex objects. For example, I consider the `QCA::TLS` implementation to be the state-of-the-art regarding the `QTimer` approach, and yet the resulting code is barely maintainable.

## The Challenge Ahead

You pull up a clean, blank source file, and declare a nice `QObject`-based API. You then set out to write an implementation for your API, and, upon completion, you discover that your object doesn't behave the way a user would expect. It seems downright silly, but the most straightforward implementations are almost entirely wrong. Did you use "signals last" or `QTimer`? No? Then you're probably in violation of at least DS and SS.

Before you attack the [Delta Object Rules][dor] for being overbearing, keep in mind that they aren't rules I pulled out of thin air. DOR states the way everyone generally expects Qt objects to behave. Yes, even *you*. You start working with someone's object, and you expect DOR-compliance. All I did was give it a name. The expectations have always been there.

So, straightforward implementations behave wrong. A non-straightforward implementation is needed for proper behavior. Can we improve this situation? Can we create convenience mechanisms so that we can write properly-behaving objects with straightforward implementations? That's the challenge ahead, and we'll get into that in [Part 2][part2].

[dor]: /2007/04/27/delta-object-rules/
[part2]: /2010/01/09/meeting-expectations-in-qobjects-part-2/

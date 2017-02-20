---
layout: post
title:  Signal Retraction
date:   2006-10-26 05:09:00
author: justin
categories: programming
---
Qt 4 introduced easier ways to queue method calls until the next eventloop cycle. Now you can connect signals and slots together with QueuedConnection. You can use the connectionless `QMetaObject::invokeMethod()` with the `QueuedConnection` mode as well. In the old days, a delayed method call took more work. Usually you would store argument values somewhere, use a QTimer to invoke a special slot at a later time, and this slot would then take the original argument values and call the method you wanted to call in the first place. With Qt 4, you can get this stuff in one line.

<!--more-->

*Anyway*, delayed calls are often a good thing. They are most useful for assisting with [delayed signals][delayed-signals] (which can also aid in making your object [signal-safe][signal-safety]). In fact, if you wanted to go overboard, you could emit all of your signals late. This would make your object pretty darn safe, just not very optimized.

There is a price to all of this delayed business though. One major issue is that the state of the object may have changed between the emit and the time of receipt. Maybe a certain object expects you to inspect one of its properties from within a slot connected to one of its signals. If you were to make this a `QueuedConnection`, it might not be possible to use this object correctly. Of course, given that the connection mode is the *user's* choice, hopefully no new objects are being written that rely on `DirectConnection`. If they are, they'd better be documented as such.

Unfortunately, there are some cases where getting the state right just isn't possible, at least not in a clean way. This is when you have object re-use or object deletion.

## Object Re-use

Take, for example, `QTcpSocket`. Imagine you set up your signals using `QueuedConnection`, then call `connectToHost("server1"...)`, `abort()`, `connectToHost("server2"...)` (in that order), and server1 is up but server2 is down. There is a race condition, whereby `connected()` might be emitted (for server1), and then `error()` (for server2), but your application receives both of these signals after calling the second `connectToHost()`. Since you cannot associate these signals with a particular "usage" of `QTcpSocket`, the assumption is that you were able to connect to server2. In fact, if it takes some time before the connection to server2 results in an error, your application might be confused for awhile.

One solution to this problem might have been to have `connectToHost` return a context id, which would then be present in any subsequent signals. This is the natural approach to an asynchronous system. Of course, this is also ugly and would defeat much of the clean fun of Qt programming, so we won't give this any further consideration. :)

Anyhow, this problem is not limited to making `QueuedConnection` signals with `QTcpSocket`. It is conceivable that `QTcpSocket` might use some queued stuff internally. Indeed, I've read the code and I've seen `QMetaObject::invokeMethod()` in there. This has the same kind of potential, where actions might be queued, and they are still in the queue even if you reset the object for a new use.

The answer? Don't use `QueuedConnection` on a stateful object (like `QTcpSocket`), and delete the object if you want to cleanly re-use it. Any queued business will die with the object. Yes, this means `abort()` is useless to you.

## Object Deletion

Deleting an object will stop any internal queued stuff.

*However* (and this is a big one), "externally" queued stuff, such as a signal/slot connection that you made using `QueuedConnection`, will not be stopped. Qt only verifies the signal and slot connection during the time of emit. When it is time for the slot to be called, only the receiver needs to still exist. This means that if you are using `QueuedConnection`, it is entirely possible that you may delete an object, only to later have one of your slots be called because the sender emitted a queued signal just before you deleted it. Fortunately, Qt does zero out the `sender()` if it no longer exists at the time of the slot.

The answer? Don't use `QueuedConnection` on a stateful object if you want to be able to delete it without it haunting you.

## Signal Retraction

Sometimes, you need to queue an action internally. We already went over the benefits. The problem is that by doing queued things, your object is not easily trusted for re-use. If someone wants to re-use your object, then they are better off deleting it and starting fresh. Can we improve the situation? Yes! With Signal Retraction.

Signal retraction is where a queued outbound signal is canceled when the object is re-used. For example, if `QTcpSocket` practiced signal retraction, then calling `abort()` would cancel all queued activities, such that we wouldn't receive the `connected()` signal from the first usage. This ensures that the signals received are safely coupled with the usage that caused them.

So how the heck do you retract a signal? By any means possible, I don't care how you do it. Go digging in the event queue and destroy the event if you want. :) Personally, I tend to use `QTimer` with a timeout of 0, and emit my signal when the `timeout()` occurs. I use a real `QTimer` object, not `singleShot()`, so that I can call `stop()` on it if my object needs to reset, thus preventing the emit.

Here's an example using QTimer:

```c++
class MyObject : public QObject
{
    Q_OBJECT
public:
    bool ready;
    QTimer readyTrigger;

    MyObject() : readyTrigger(this)
    {
        ready = false;
        connect(&readyTrigger, SIGNAL(timeout()), SLOT(ready_timeout()));
        readyTrigger.setSingleShot(true);
    }

    // kicks the object into motion.  emit ready() when ready.
    void start()
    {
        ready = true;
        readyTrigger.start();
    }

    // reset the object to the initial state.  no ghost signals.
    void reset()
    {
        ready = false;
        readyTrigger.stop();
    }

signals:
    void ready();

private slots:
    void ready_timeout()
    {
        emit ready();
    }
};
```

There's a little bit of [delayed signals][delayed-signals] philosophy going on in there, if you're confused about the way it is written.

Happy coding!

[signal-safety]: /2008/02/04/signal-safety-revised/
[delayed-signals]: /2006/04/14/delayed-signals/

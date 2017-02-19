---
layout: post
title:  Meeting expectations in QObjects (Part 2)
date:   2010-01-10 02:26:00
---
In [Part 1][part1], we discussed how QObjects often don't behave the way we expect. This is largely because developing a QObject that behaves the way we would expect is not always a simple process. Commonly, if you develop a QObject in the most straightforward manner, then it will behave unexpectedly, even to yourself!

In this article we'll talk about a tool that can assist with making our objects behave properly: `ObjectSession`. It is just one of many tools I've come up with over the years, but I'm starting to feel confident enough in it that I figure it is time to talk about it. :)

<!--more-->

## Introducing ObjectSession

The great thing about `ObjectSession` is that the concept is quite simple and yet it manages to solve many problems. The way it works is you create an `ObjectSession` instance that keeps track of a "session" of queued signal/slot activity. Instead of using `QMetaObject::invokeMethod()` or `QTimer` to perform queued calls directly, you tell your `ObjectSession` to perform these calls, with the advantage that you can reset the session at any time to cause all queued calls to be canceled.

Additionally, there is `ObjectSessionWatcher` which can keep track of session validity. You know the old trick about using `QPointer` on the stack, to find out if your object has been deleted after you emitted a signal? `ObjectSessionWatcher` is intended to replace this usage. You create it on the stack in the same way as `QPointer`, but you use it to find out if your session has been reset instead of if your object has been deleted. This allows you to easily provide signal safety for all methods, not just destructors.

`ObjectSession` can currently be found under src/irisnet/corelib in [Iris][iris]. Here is a snippet from `objectsession.h`:

```c++
class ObjectSession : public QObject
{
    Q_OBJECT

public:
    ObjectSession(QObject *parent = 0);
    ~ObjectSession();

    void reset();
    bool isDeferred(QObject *obj, const char *method);
    void defer(QObject *obj, const char *method,
        QGenericArgument val0 = QGenericArgument(),
        QGenericArgument val1 = QGenericArgument(),
        [...]);

    [...]
};

class ObjectSessionWatcher
{
public:
    ObjectSessionWatcher(ObjectSession *sess);
    ~ObjectSessionWatcher();

    bool isValid() const;

    [...]
};
```

There are a few more methods available than shown above, but these are the main ones.

Now let's go over some usages.

## Basic setup

Usually you'll have just one ObjectSession per object, set up like such:

```c++
class Foo : public QObject
{
    Q_OBJECT

private:
    ObjectSession sess;

public:
    Foo(QObject *parent = 0) :
        QObject(parent),
        sess(this)
    {
    }
};
```

Note: The `defer()` call allows you to make queued calls on any object, so it is not necessary for the ObjectSession to live in the "public" object like this. You can easily tuck it away in a private class d-pointer if you want, and still target signals of the public class.

## Emitting a delayed signal, with ability to retract

The `defer()` call works just like `QMetaObject::invokeMethod()`, even allowing you to specify arguments:

```c++
void Foo::startServer()
{
    udp = new QUdpSocket(this);
    if(!udp->bind())
    {
        // app expects the error() signal to be asynchronous
        sess.defer(this, "error", Q_ARG(Foo::Error, ErrorBind));
        return;
    }

    [...]
}

void Foo::abort()
{
    sess.reset();
}
```

Now the user can call `startServer()`, `abort()`, `startServer()` all in a row without returning to the event loop and there is no worry that the above signal might accidentally be misinterpretted as part of the second "session". The `sess.reset()` method destroys the deferred call queue.

## Safely emitting a signal in the middle of a code block

Instead of using `QPointer` to check if your object is destroyed, use `ObjectSessionWatcher` to see if your object state is still sane.

```c++
void Foo::tcpSocket_connected()
{
    [...]

    ObjectSessionWatcher watch(&sess);
    emit connected();
    if(!watch.isValid())
        return;

    tcpSocket->write("hello world");

    [...]
}

void Foo::abort()
{
    sess.reset();
    tcpSocket->deleteLater();
    tcpSocket = 0;
}
```

Now if you emit `connected()` and the user calls `abort()` during that slot, the watcher is immediately notified that the session is no longer valid. This way we can bail out instead of crashing on the next line.

Note that destructing `ObjectSession` also causes any watchers to be invalidated, so we get signal-safety of the destructor for free too.

## Emitting multiple signals in a row

There are a couple of ways to do this. One way is to use deferred signals for all but the first:

```c++
void Foo::someFunction()
{
    [...]

    sess.defer(this, "sig2");
    emit sig1();

    [...]
}
```

The other way is to use a watcher:

```c++
void Foo::someFunction()
{
    [...]

    ObjectSessionWatcher watch(&sess);
    emit sig1();
    if(!watch.isValid())
        return;
    emit sig2();

    [...]
}
```

Which way you do it depends on your mood. The second method is a little more straightforward, but has the drawback that if the user invokes a nested eventloop in response to `sig1()`, then `sig2()` will not be emitted during the new eventloop. So for example if `sig1()` causes a modal dialog to appear, then the object might seem like it is blocking or not working until the dialog is dismissed.

(That said, nested eventloops are [pure evil][ne-evil] and will cause you all sorts of problems anyway. Can you imagine other events suddenly activating in your object, while `someFunction()` is still sitting there partway executed? Maybe even causing other signals to emit, before `sig2()` ? Yeeeesh..)

## Deferring calls to your own slots

In all the examples so far we've used `defer()` to invoke signals, but it is also useful for queuing up calls to your own slots. Here's an alternate way to handle the delayed signal from earlier:

```c++
void Foo::startServer()
{
    // set object state
    state = Starting;

    udp = new QUdpSocket(this);
    if(!udp->bind())
    {
        // process the error later, so object state remains consistent
        sess.defer(this, "slot_bindError");
        return;
    }

    [...]
}

int Foo::state() const
{
    return state;
}

void Foo::slot_bindError()
{
    // now we atomically change the state and emit error
    state = Inactive;
    emit error(ErrorBind);
}

void Foo::abort()
{
    sess.reset();
}
```

There is also `deferExclusive()` to ensure that a call is only queued once at a time. This is useful for making sure an "update"-style function runs at the next eventloop.

```c++
void Foo::sendText(const QString &str)
{
    queueOutboundData(str);

    // make sure we push out pending data at the next eventloop
    sess.deferExclusive(this, "doSend");
}

void Foo::sendPhoto(const QPixmap &p)
{
    queueOutboundData(p);

    // make sure we push out pending data at the next eventloop
    sess.deferExclusive(this, "doSend");
}

void Foo::doSend()
{
    // push all items in the queue out the network
    [...]
}
```

If the session is reset, such deferred slot calls would get canceled in the same way that deferred signals do. How convenient! Now you can avoid having background slots continuing to execute when they shouldn't be.

## Conclusion

`ObjectSession` is a simple class that can be used to solve a number of problems with QObject development that are otherwise quite awkward to solve. The key for me was to discover that almost all of the problems have to do with an object switching states, stopping, or being destroyed. So, if there could be a simple way to manage those situations then we could solve just about everything. I'm using `ObjectSession` in a few of my classes now and I'm quite pleased at how easy it is.

Happy coding!

[part1]: /2008/08/28/meeting-expectations-in-qobjects-part-1/
[iris]: https://github.com/psi-im/iris
[ne-evil]: /2008/11/22/nested-eventloops-are-pure-evil/

---
layout: post
title:  Signal Safety Revised
date:   2008-02-05 06:11:00
---
It has been almost four years since I first wrote about Signal Safety. The concept has held up well all this time, and I follow it (as well as the rest of the [Delta Object Rules][dor]) in any new code I write. However, last year I discovered there is more to the problem, and given that the original article was written in the Qt 3 era I decided it was time for an update. The original article is even obsolete, now that Qt 4 restricts `QObject::deleteLater()` processing to the same event-loop depth. This article shall serve as a modern replacement for the original article.

<!--more-->

## The Problem

Most method calls in a program run in a self-contained or "black box" fashion. That is, a method is given a set of inputs, and, upon completion, that method provides some expected outputs. The state of the universe *during* the method call is most often undefined. This goes for plain C functions as well as for C++ class member functions.

Suppose you are calling `QString::truncate(int)`, which shortens a QString to some length. Now, suppose you could somehow interrupt the `truncate()` function while it is in progress, and proceed to call `QString::length()` on the same object. What value would you expect `length()` to return? Would it be the length before truncation? After truncation? Some value in between? Maybe the program would simply crash, which is certainly within its rights. After all, QString makes no claim about the integrity of its internal data structures while the `truncate()` method is being executed. The object state is really none of your business until the method completes, at which point QString is ready to take on your next method call.

Of course, this should all go without saying. Unless the author of an object claims some degree of thread-safety or such, the assumption is that you can't poke around at an object while it is busy. However, this becomes much more complicated with the introduction of callbacks. With a callback, an object or method makes a call "back" to you. Qt signals are a form of callbacks. You connect an object's signal to one of your slot methods, and now that object will call your slot methods. With callbacks, the million dollar question is: What is the state of the object that is invoking the callback, during the callback? That is, are any claims made about the integrity of the object's internal data structures? What methods are you allowed to call on this object during the callback?

In the world of Qt, the answer is most often: "Who knows?" Signals are emitted wherever the author finds most convenient to do so. No consideration is made about the state of the object during these emits. This means that most objects are in an undefined state during signal emission. Calling a method on an object during that same object's signal emit is the same in concept as interrupting `QString::truncate()` and calling `length()` on that same string. In both cases, a method is randomly interrupted, and you are accessing the object in a state that was likely not considered by the author.

A real-world example I mentioned in one of my other articles is a bug/problem I found in `QTcpSocket` some months back (note: it could be corrected by now, but I have not checked). On failure, this object emits the `error()` signal. During this signal, calling `QTcpSocket::read()` will result in a crash. Note that calling `read()` on a failed `QTcpSocket` object is absolutely allowed. This is because `QTcpSocket` buffers incoming data, and you are allowed to read this data at any time, even after the underlying TCP connection is closed. However, `QTcpSocket` is in an inconsistent state when the `error()` signal is emitted. It is likely in a state of mid-cleanup, and not expecting any method calls. Once `QTcpSocket` has finished what it is doing and returned to the event loop, you can safely call `read()`. So who is at fault here? Is it me for assuming I can call `read()` when I'm really not supposed to? Or is it Qt, for having a bug that prevents `read()` from working in a situation that it clearly should have?

The core of the problem is that nothing is written in a stone tablet saying what we are supposed to do here. If I had to describe the current state of things, it would be that all methods are implied to be callable during a signal, minus the destructor. Yet, code is generally not written to match this implicit contract. Sure, the code works most of the time, but it is only sheer luck, and sooner or later we get a problem like the above `QTcpSocket` issue.

## The Solution

In my old article, I proposed that you should be allowed to call an object's destructor during its signals. In this article, I expand the definition to include **all** methods of the object.

Imagine an object that doesn't use callbacks. You may call members of this object:

```c++
foo.init();
foo.addData(someData);
foo.sort();
foo.reset();
```

After each of these member calls, foo returns to a 'neutral' state whereby you are free to call any other method. This is within reason, of course. Maybe there's one member that you aren't allowed to call unless you make some other member call first. That's all fine. It is still the neutral state.

Essentially, I propose that an object should be in the neutral state during a signal emit. Here's a rather contrived sample class:

```c++
class TextHolder : public QObject
{
    Q_OBJECT
private:
    QString *curText;
    bool modified;

public:
    TextHolder()
    {
        curText = new QString("");
        modified = false;
    }

    ~TextHolder()
    {
        delete curText;
    }

    QString text() const
    {
        return *curText;
    }

    void setText(const QString &text)
    {
        delete curText; // delete the text we were currently holding
        emit textChanged(text); // announce the new text we received
        curText = new QString(text); // take this new text
        modified = true; // flag that this object was modified
    }

signals:
    void textChanged(const QString &newValue);
};
```

Alright, so in the object's neutral state, you are allowed to call `text()`, because `curText` is always non-NULL and valid in that state. We can see here that during the `textChanged` callback, `curText` is invalid. Therefore, this code violates signal safety. Calling `text()` during the callback is dangerous. Not only does this object need to be in the neutral state before and after the call to `setText`, it needs to be in the neutral state during `textChanged`. Let's fix it by ensuring `curText` is atomically updated:

```c++
void TextHolder::setText(const QString &text)
{
    emit textChanged(text);
    delete curText;
    curText = new QString(text);
    modified = true;
}
```

We could also do it before the signal:

```c++
void TextHolder::setText(const QString &text)
{
    delete curText;
    curText = new QString(text);
    emit textChanged(text);
    modified = true;
}
```

Whether `curText` is modified before or after the `textChanged()` signal depends on whatever you think makes more sense for the class. The point is that calling `text()` during the textChanged() signal is made legal, because it would have been legal to call it while not in the signal.

There is another issue to address though, which is to make the destructor callable. Right now, if the `TextHolder` object is deleting during the `textChanged()` signal, then the application will probably crash when the callback completes and there is an attempt to access the `modified` bool which no longer exists. Let's fix that:

```c++
void TextHolder::setText(const QString &text)
{
    delete curText;
    curText = new QString(text);
    modified = true;
    emit textChanged(text);
}
```

Now whenever you are able to call methods on `TextHolder`, you can be sure it is in the neutral state. There is one catch, though: you cannot call `setText()` from within the callback, else you get infinite recursion. Oh well, at least that one should be expected.

## Objects on a Signaling Course

It is not uncommon for an object to need to emit multiple signals in one pass of the event loop. For example, suppose you have an object that receives lines of text over the network, and emits a signal for each line of text received:

```c++
void LineParser::receiveData(const char *data)
{
    QStringList lines;
    ...
    // [ parse data into lines ]
    ...
    foreach(const QString &line, lines)
        emit lineReceived(line);
}
```

Now, suppose there is a function called `LineParser::reset()`, which clears all buffers and initializes the `LineParser` object back to what it would have been at original construction. What happens when you do the following?

```c++
void MyApp::slot_lineReceived(const QString &line)
{
    if(!lineUnderstood(line))
    {
        lineParser->reset();
        ...
        return;
    }
    ...
}
```

Guess what? `MyApp` could still get fed a bunch of lines it doesn't care about. We can solve this by adding a flag in `LineParser` that accounts for whether or not the object is in a reset state. Let's say there's a bool called `isReset`, that is set to true on construction and also after a call to `reset()`. We can then rework the `receiveData()` function:

```c++
void LineParser::receiveData(const char *data)
{
    QStringList lines;
    ...
    // [ parse data into lines ]
    ...
    foreach(const QString &line, lines)
    {
        emit lineReceived(line);
        if(isReset)
            break;
    }
}
```

Now `LineParser` can be reset in the middle of the line signaling, living up to what `MyApp` had expected. Of course, we still can't delete the `LineParser` object in the `lineReceived()` signal. Let's fix that too:

```c++
void LineParser::receiveData(const char *data)
{
    QStringList lines;
    ...
    // [ parse data into lines ]
    ...
    QPointer<QObject> self = this;
    foreach(const QString &line, lines)
    {
        emit lineReceived(line);
        if(!self)
            break;
        if(isReset)
            break;
    }
}
```

Yep, it's that same old trick, just like the stuff with Qt 3's `QGuardedPtr` in the old article.

## The Case for Safe Deletion

At its core, `QObject` supports being deleted while signaling. In order to take advantage of it you just have to be sure your object does not try to access any of its own members after being deleted. However, most Qt objects are not written this way, and there is no requirement for them to be either. For this reason, it is recommended that you always call `deleteLater()` if you need to delete an object during one of its signals, because odds are that `delete` will just crash the application.

Unfortunately, it is not always clear when you should use `delete` vs `deleteLater()`. That is, it is not always obvious that a code path has a signal source. Often, you might have a block of code that uses `delete` on some objects that is safe today, but at some point in the future this same block of code ends up getting invoked from a signal source and now suddenly your application is crashing. The only general solution to this problem is to use `deleteLater()` all the time, even if at a glance it seems unnecessary.

I commonly offer a `reset()` function in my objects. However, I have to be careful about the code path at the time `reset()` is called. For example:

```c++
class NetworkObject
{
    Q_OBJECT
private:
    QTcpSocket *sock;

public:
    NetworkObject() : sock(0) { }
    ~NetworkObject()
    {
        reset();
    }

    void reset()
    {
        delete sock;
        sock = 0;
    }

signals:
    void error();

private slots:
    void sock_error(QAbstractSocket::Error)
    {
        reset();
        emit error();
    }
};
```

With the above object, the `reset()` function might be called in three different places:

1. Directly by the user when the object is in a neutral state.
2. When the object destructs.
3. When the socket encounters an error.

The above class is broken, of course, because it deletes `QTcpSocket` while inside the `QTcpSocket::error()` signal. How can we solve this problem? Well, we could rework `sock_error()` like this:

```c++
void NetworkObject::sock_error(QAbstractSocket::Error)
{
    // the disconnect statement is needed to prevent signals between now and actual delete
    sock->disconnect(this);
    sock->deleteLater();
    sock = 0;
    emit error();
}
```

However, this sort of violates the DRY (Don't Repeat Yourself) principle. Right now our `reset()` function is small, but if it starts growing large then there will be a lot more to repeat. Let's rework the `reset()` function instead:

```c++
void NetworkObject::reset()
{
    sock->disconnect(this);
    sock->deleteLater();
    sock = 0;
}
```

There, now we only reset in one place. Unfortunately, in two out of the three situations `reset()` could be called, only one of them actually needs the `deleteLater()`. This seems wasteful. What can we do? How about this:

```c++
class NetworkObject
{
    Q_OBJECT
private:
    QTcpSocket *sock;

public:
    NetworkObject() : sock(0) { }
    ~NetworkObject()
    {
        reset();
    }

    void reset(bool delLater = false)
    {
        if(delLater)
        {
            sock->disconnect(this);
            sock->deleteLater();
        }
        else
            delete sock;
        sock = 0;
    }
    ...
signals:
    void error();

private slots:
    void sock_error(QAbstractSocket::Error)
    {
        reset(true);
        emit error();
    }
};
```

There, now we get exactly what we want, although I wonder if this approach may become risky over time. Perhaps it is best to have `reset()` always do `deleteLater()`, just in case the object gets so complex that it is unclear what value you need to pass to reset. It also gets more complex if there are other `QObjects` to delete. If `delLater` does `deleteLater()` on every object, then that's not optimized. It should only use `deleteLater()` on the one object that matters. Maybe reset could be rewritten like this:

```c++
void NetworkObject::reset(QObject *delLater)
{
    ...
    // [ delete all objects, except for whichever one is delLater ]
    delLater->disconnect(this);
    delLater->deleteLater();
    ...
}
```

As you can see, there's a lot of hassle that can go into ensuring correct usage of `deleteLater()` vs `delete`.

A simpler approach is to just use `deleteLater()` all the time. Unfortunately, this is just not standard practice among Qt developers. They will still try to guess which one they should use based on the observable code path, thinking that `deleteLater()` is unoptimized and should only be used as a last resort. It is error-prone to guess here, because you can't always predict how that code path will be invoked. My advice is to use `deleteLater()` every time, unless you *know* you are using an object that can be deleted in its signals (in which case you can `delete` every time for that object).

You are doing a great favor by designing your object to be deleted while in a signal. If your object can handle a `reset()`-style call while it is "on a signal course", then you've already done the heavy lifting. At that point, handling a destructor is not much extra effort.

## Conclusion

We have a problem today, where we are mucking with objects while they are in an undefined state. At least with `delete` we can choose to use `deleteLater()` instead. But what about the problem of calling `QTcpSocket::read()` while in the `error()` signal? Call `readLater()`? There is no such thing!

I suppose to be absolutely safe, we should avoid making any call to an object during one of its signals. In my opinion, this is simply not practical with the way all of us write Qt code today. A better approach is to fix the implementation of our objects, so that they behave the way our users (and ourselves!) had always expected them to behave.

[dor]: /2007/04/27/delta-object-rules/

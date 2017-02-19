---
layout: post
title:  Nested Eventloops
date:   2006-10-24 06:46:00
---
To define the title: a nested eventloop is when you invoke the eventloop again rather than returning back to it. This is `QCoreApplication::processEvents()`, `QDialog::exec()`, `QMessageBox::information()`, and the like. In all but the most controlled situations, performing a nested eventloop using these traditional methods is dangerous and should be avoided. On the other hand, restricting event processing to a specific subset of objects can be safe. Unfortunately, Qt doesn't provide a direct and general way to perform this kind of "scoped" eventloop.

<!--more-->

First, let's discuss why the traditional nested eventloop functions are bad. Spinning the eventloop is a big deal. Every object in the application (or, more specifically, in the thread, if your app is multithreaded, but most aren't) will run when the eventloop is run. This means event handlers being called, possible signals emitted as a result of those events, etc. This can cause unpredictable behaviors. If the application has not "returned to the eventloop" yet, then there may be functions on the stack that are incomplete, and so the entire state of the program is not ready to receive events yet. A form of re-entrancy is also possible, whereby an event handler is called, sometime down the line the eventloop is run again, and then the same event handler is called again. The event handler is now on the stack twice. Did you write your application to work under this kind of condition?

Here is an example of an object that prompts the user for two socket ids, and then emits a signal when either one has data. This is a bit contrived, but it is short, and the problem should be easy to spot.

```c++
class MyObject
{
    QSocketNotifier *sn1, *sn2;

    void setup()
    {
        int sockfd1 = get_next_sockfd();
        sn1 = new QSocketNotifier(sockfd1, QSocketNotifier::Read);
        connect(sn1, SIGNAL(activated(int)), SLOT(sn_activated(int)));

        int sockfd2 = get_next_sockfd();
        sn2 = new QSocketNotifier(sockfd2, QSocketNotifier::Read);
        connect(sn2, SIGNAL(activated(int)), SLOT(sn_activated(int)));
    }

    int get_next_sockfd()
    {
        return QInputDialog::getInteger(0, "Prompt", "Socket FD:");
    }

signals:
    void dataReady();

private slots:
    void sn_activated(int)
    {
        delete sn1;
        delete sn2;
        emit dataReady();
    }
};
```

Do you see it? `QInputDialog::getInteger()` spins the eventloop. If the first socket has data to read, the second call to getInteger might cause `sn_activated(sockfd1)` to be called. If this happens, the program will likely crash at the `delete sn2;` line. Now, one obvious fix would be to set `sn2` to zero at the top of `setup()`, but that's not the point of this discussion. The point is that the use of `getInteger()` may mean you need to take additional precautions with your code, that you normally wouldn't have to do. This was a simple example. In a more complex, real-world situation, you may not know which function is spinning a nested eventloop, and the fixes may not be as straightforward.

In simple applications, running a nested eventloop is not a big deal. If you aren't running any non-GUI objects, then feel free to use `QMessageBox::information()`. It is convenient, and modality usually protects you from re-entrancy. However, if you *do* have non-GUI objects, then `QMessageBox::information()` is a potential disaster. Suppose you have slots listening for network data. Those slots might be called, and your application may not be written to work properly in such conditions.

Ultimately, what this means is that nested eventloops can be safe if you know they will be safe in a given situation. But if you don't know that they will be safe, then you cannot use them. And if you are a library author, you're in a pickle, because you cannot know if the application will be safe. **Please, if you are writing a library, DO NOT use processEvents(), QMessageBox::information(), etc.** If you absolutely must use a nested eventloop in a library, any method leading to that effect should be clearly documented. This way, the application developer can either avoid the function, or design his application to withstand the effects.

You might say that problems with nested eventloops are the result of poor programming at the higher level. That is, the application should be designed to "withstand the effects" of any method potentially running a nested eventloop. This is one of Trolltech's positions (as of the time of this writing, October 23rd 2006, which I may talk more about later). Anyway, I'm here to tell you that you cannot write such a generic application. What if `QString::operator+()` invoked a nested eventloop? Good luck trying to make your application withstand such a thing, even if you were told about it up front. But having to account for any random method invoking a nested eventloop? Not possible. It would be like thread programming without mutexes.

We need assurances, as programmers, of what functions are going to do. We need to know what global variables they might modify (consider functions that return static data). We need to rely on their signatures being correct at runtime (return value, arguments). If we cannot have any expectations about the methods we call, then it would not be possible to write programs. If a method may run a nested eventloop, the user of the method **must** be informed about it.

In short, avoid using nested eventloops. This concept joins [Signal Safety][signal-safety] and [Delayed Signals][delayed-signals] as another essential Qt programming practice for clean and predictable code.

[signal-safety]: /2008/02/04/signal-safety-revised/
[delayed-signals]: /2006/04/14/delayed-signals/

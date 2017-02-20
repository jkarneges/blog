---
layout: post
title:  Delayed Signals
date:   2006-04-14 12:59:00
author: justin
categories: programming
---
Delayed signals are a design mechanism for reducing errors by people using your object (similar in spirit to [Signal Safety][signal-safety]). This is a difficult topic, because up until this writing I still haven't been able to formally define a universal rule for when you should use delayed signals. I'll attempt that now, and show some code to illustrate.

<!--more-->

**Rule:** *When requesting a result from an object by first calling a member function and then obtaining the result via a signal, the signal should not be emitted until after the application has returned to the event loop.*

To explain just what the heck that means, consider how signal usage can fall into one of two groups: 1) as a means of returning or indicating a result of a requested operation, or 2) as a means of announcing general status changes, independent of a request. #1 is used in worker objects, for example `QTcpSocket` or `QProcess`, where there is a request/response expectation. #2 is used predominantly in GUI objects (widgets), where the signals that occur are not requested nor required to operate.

My rule only pertains to #1. One good way to ask yourself if your signal falls under the rule is if it appears to play a part in returning an answer. Synchronous methods will use the return value to deliver an answer. Asynchronous methods don't use the return value for the answer, and instead give or indicate the answer via a signal. If this is you, then you should use delayed signals.

Now, with that out of the way, you might ask: why use delayed signals?

Well, first of all, you probably wouldn't be using a signal if you didn't expect your response to take time to generate. If your object uses the event loop to do its processing before signalling a result, then your signals will already be "delayed" in the sense I'm talking about.

The problem arises when your object knows the result early, and therefore the object does not need to run the event loop in order to generate the result. If the signal is emitted before returning to the event loop (for example, if it is emitted during the "request" member function), then this would not qualify as delayed.

If your object is unpredictable, that is it sometimes emits delayed and other times non-delayed, then the user of your object may have to write extra code to handle both possibilities. The whole point of using a signal is to be able to provide an asynchronous reply, so if we want to pick just one pattern then "delayed" is our answer. It is the more natural pattern for signals anyway.

If you aren't convinced, the best explanation is with code. Suppose there's an object that downloads a file from a URL:

```c++
class Downloader : public QObject
{
    Q_OBJECT
public:
    void start(const QString &url);
    void stop();

signals:
    void finished(bool success);
};
```

Simple enough. It will emit `finished(true)` if the file download succeeded, or `finished(false)` if there is a problem.

Now let's make a simple app that tries to download two files, and immediately quits if there is a problem:

```c++
class App : public QObject
{
    Q_OBJECT
public:
    Downloader file1, file2;
    int done;

    void start()
    {
        connect(&file1, SIGNAL(finished(bool)), SLOT(dl_finished(bool)));
        connect(&file2, SIGNAL(finished(bool)), SLOT(dl_finished(bool)));

        printf("Starting downloads\n");
        done = 0;
        file1.start("http://example.com/document.txt");
        file2.start("http://example.com/otherDocument.txt");
    }

signals:
    void quit();

private slots:
    void dl_finished(bool success)
    {
        if(success)
        {
            ++done;
            if(done == 2)
            {
                printf("Downloads success.\n");
                emit quit();
            }
        }
        else
        {
            file1.stop();
            file2.stop();

            printf("Error during downloads!\n");
            emit quit();
        }
    }
};
```

Looks good, yes?

Actually, no, it isn't quite right. We assumed that signals would always be emitted to us delayed. If `Downloader` doesn't emit delayed 100% of the time, then we have bugs.

Let's assume `Downloader` may or may not always emit delayed. We'll write extra code then, to handle all situations:

```c++
class App : public QObject
{
    Q_OBJECT
public:
    Downloader file1, file2;
    int done;
    bool error;                          // added

    void start()
    {
        connect(&file1, SIGNAL(finished(bool)), SLOT(dl_finished(bool)));
        connect(&file2, SIGNAL(finished(bool)), SLOT(dl_finished(bool)));

        printf("Starting downloads\n");
        done = 0;
        error = false;                   // added
        file1.start("http://example.com/document.txt");
        if(!error)                       // added
            file2.start("http://example.com/otherDocument.txt");
    }

signals:
    void quit();

private slots:
    void dl_finished(bool success)
    {
        if(success)
        {
            ++done;
            if(done == 2)
            {
                printf("Downloads success.\n");
                emit quit();
            }
        }
        else
        {
            file1.stop();
            file2.stop();
            error = true;                // added

            printf("Error during downloads!\n");
            //emit quit();               // removed
            QTimer::singleShot(0, this, SIGNAL(quit()));   // added
        }
    }
};
```

If `finished(false)` is emitted non-delayed by `file1.start()`, then this means we will perform our deinitialization process before `App::start()` completes. We use the `error` bool to track if this happens, so that we can skip the calling of `file2.start()` as necessary. Since `file2.stop()` would occur before the call to `file2.start()` (had we not fixed the code), this would most likely mean that the second file transfer wouldn't be stopped at all.

Finally, if we do a normal emit of `quit()` during a non-delayed call to `finished()`, then we lose signal-safety. If `App` is deleted during the slot that listens to `quit()`, the code following `file1.start()` will cause a crash when `App`'s members are accessed (such as the `error` bool). We can give ourselves signal-safety by doing a delayed emit of `quit()` with `QTimer::singleShot`.

Bottom line: there's less code if we can assume that `Downloader` always uses delayed signals.

[signal-safety]: /2008/02/04/signal-safety-revised/

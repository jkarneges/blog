---
layout: post
title:  Delta Object Rules
date:   2007-04-28 00:05:00
---
The Delta Object Rules (DOR) are a set of rules that QObjects should obey in order to ensure predictable and safe behavior. The rules are:

* [Signal Safe][signal-safety] (SS). It must be safe to call methods on the object, including the destructor, from within a slot being called by one of its signals.
* [Delayed Signals][delayed-signals] (DS). For request/response, response signals must not occur in the same eventloop cycle as the request.
* [Nested Eventloops][nested-eventloops] (NE). Nested eventloops must not be used within the same thread as the object.
* [Signal Retraction][signal-retraction] (SR). For request/response, response signals must not be emitted if the request has been cancelled (deleting the object also qualifies as cancellation).

<!--more-->

Some objects in Qt itself are known to be DOR-compliant, despite not being labeled in any special way. These are: `QObject` (except for the `destroyed()` signal), `QTimer`, and `QSocketNotifier`. You can assume these objects are compliant, but do not assume this for any other objects (in any project, Qt or otherwise) unless they are labeled. **Update (2008/8/27):** As of Qt 4.4, `QTimer` and `QSocketNotifier` will print console errors if destructed during their signals. Internally they are actually safe to destruct, but since Trolltech is trying to deprecate this feature these objects are to no longer be considered DOR-compliant.

The DOR label can be applied to a method, an object, or even an entire project. Exceptions are allowed. For example, it would be fine to say an object is DOR compliant except for one signal that is not safe to delete from (SS not supported for that signal) or for one method that invokes a nested eventloop (NE not supported for that method).

When using an object that has no compliancy labeling, it must be assumed that the object does not conform to any of the above rules. SS and DS are possible to work around if they are not supported. SR is harder. NE is not possible. For NE, you may want to inspect the object source code (or ask the author) if there is a concern about it.

[signal-safety]: /2008/02/04/signal-safety-revised/
[delayed-signals]: /2006/04/14/delayed-signals/
[nested-eventloops]: /2006/10/23/nested-eventloops/
[signal-retraction]: /2006/10/25/signal-retraction/

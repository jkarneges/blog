---
layout: post
title:  Nested eventloops are pure evil
date:   2008-11-23 02:26:00
---
From the [gst_bus_poll() documentation of GStreamer][gst-bus-poll]:

*"This function will run a main loop from the default main context when polling.*

*You should never use this function, since it is pure evil. This is especially true for GUI applications based on Gtk+ or Qt, but also for any other non-trivial application that uses the GLib main loop. As this function runs a GLib main loop, any callback attached to the default GLib main context may be invoked. This could be timeouts, GUI events, I/O events etc.; even if `gst_bus_poll()` is called with a 0 timeout. Any of these callbacks may do things you do not expect, e.g. destroy the main application window or some other resource; change other application state; display a dialog and run another main loop until the user clicks it away. In short, using this function may add a lot of complexity to your code through unexpected re-entrancy and unexpected changes to your application's state."*

See, I'm not the only crazy one around. Now, run along and get rid of those QMessageBoxes.

[gst-bus-poll]: http://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstBus.html#gst-bus-poll

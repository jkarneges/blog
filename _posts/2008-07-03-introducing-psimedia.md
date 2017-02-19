---
layout: post
title:  Introducing PsiMedia
date:   2008-07-03 21:16:00
---
<a href="/assets/psimedia2.png">![PsiMedia Test screenshot](/assets/psimedia2_small.jpg)</a>

Voice (and video) chat is a feature we've wanted in Psi for a long time. However, implementing voice/video chat is not straightforward, and this is partly due to all of the new concepts that have to be introduced into the application in order to make it happen. Cameras, microphones, codecs, and RTP are all just very foreign to Psi. The code necessary to handle a multimedia "stack" could easily exceed the amount of code in our own IM stack! Fortunately, there are libraries out there to handle the task.

<!--more-->

In 2004, we considered RealNetworks' [Helix framework][helix]. For receiving content, we found this framework to be quite mature. However, for transmitting content, it was clearly not designed for end-user desktop applications and was even GPL-incompatible in that scenario. Quite some work went into the Psi+Helix effort, but ultimately it was abandoned.

In 2005, we considered Google's [libjingle][libjingle]. We managed to get voice chat working with it, but the code never went beyond the experimental stage. This was due to the limited platform support at the time (Linux audio only at first, though Remko managed to add in Mac audio support) and libjingle's lack of maintenance. Libjingle works as a black box, handling not only multimedia but also the Jingle protocol. Unfortunately, this meant that as the Jingle protocol changed, libjingle fell out of spec. We also felt it was a tad intrusive for libjingle to be handling XMPP stuff.

In 2006, we investigated [GStreamer][gstreamer]. This framework has proved to be the most interesting thus far, for a number of reasons. Unlike the limited libjingle black-box, GStreamer is a comprehensive and flexible multimedia framework, similar in nature to Helix. It goes further than Helix though, by offering a better API for transmitting, by being GPL-compatible throughout, and by being easier to extend. I feel confident we can accomplish everything we need with GStreamer.

Today there is [Phonon][phonon], however it lacks input and transmission facilities at this time. We will keep an eye on it for the future. There is also [Farsight][farsight], which integrates with GStreamer. We may make use of Farsight, depending on our needs.

In any case, I've started a new "wrapper" project called PsiMedia. The goal of PsiMedia is to offer an API designed for the purpose of adding voice and video chat to Psi or a Psi-like client. All of the details the client does not care about will be hidden behind PsiMedia. It solves only the multimedia aspects, and not Jingle/XMPP, as I consider these two problems to be orthogonal. Currently PsiMedia wraps GStreamer, but the requirements are abstract enough that the client should not care what is actually wrapped. PsiMedia can be considered the successor of the old "Media" module I started in 2004, to wrap Helix.

Below are the requirements of the system.

What PsiMedia does:

* Tell you what audio and video devices are available.
* Tell you what audio/video modes are possible (codecs, sample rates, video resolutions, etc).
* Allow you to specify your desired modes, and the modes of the remote party, to arrive at a list if common modes.
* Capture audio/video and encode as RTP into a series of QByteArrays.
* Accept QByteArrays containing RTP, and playback any audio/video contained within.
* Play back video in a QWidget.
* Allow displaying video currently being captured (preview of yourself).
* Volume controls.
* Ability to separate the backend into a plugin, so that no new compile-time dependencies are introduced to Psi.

(RTP, by the way, is a standard packet format for transporting multimedia data in real-time. It is used by SIP, Jingle, and, well, everybody.)

What PsiMedia does not do:

* Use the network.
* Implement Jingle or anything XMPP.
* Expose anything more than very basic multimedia details. There are no filters, no pipelines, etc.

In short, PsiMedia should make implementing voice/video chat in Psi straightforward.

[helix]: http://www.helixcommunity.org/
[libjingle]: http://code.google.com/apis/talk/libjingle/index.html
[gstreamer]: http://www.gstreamer.net/
[phonon]: http://phonon.kde.org/
[farsight]: http://farsight.freedesktop.org/

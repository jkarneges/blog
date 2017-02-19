---
layout: post
title:  QCA and Smartcards
date:   2007-08-21 10:05:00
---
In public key cryptography, it is critical to keep your private key, well, *private*. One of the major features of QCA 2.0 is support for smartcard devices, and these devices allow convenient and safe storage of private keys. Smartcards perform cryptographic operations in hardware, never revealing the keys to the requesting computer. Traditionally, smartcards have a "credit card" form-factor and require a reader device, but today there also exist smartcards which have the form-factor of a USB mass storage device (they are not "cards" at all). The term "smartcard" can apply to either kind of device, as can "smart token" or simply "token".

<!--more-->

Since QCA has a pluggable backend system, changing out software algorithms for hardware calls is easily possible by writing a proper plugin. However, there is much more to using smartcards than simply substituting a private key call. Smartcards may not always be present in the computer, and may contain many stored keys. QCA offers complete smartcard support, allowing you to discover available smartcards, track inserts and removals, enumerate keys on the devices and prompt for PINs, all using a friendly Qt-based API.

The QCA distribution contains an example application called CMS Signer. It exists mainly to demonstrate using keys from smartcards in a GUI application, although it also works with on-disk keys. CMS, or Cryptographic Message Syntax, is a standard format for storing various types of secure payloads. In the case of the CMS Signer application, the CMS format is used to store digital signature data created from X.509 certificates (fun fact: S/MIME email consists of CMS-formatted parts wrapped in MIME). It is possible to configure and use smartcards completely within the CMS Signer GUI, and I'll demonstrate this below.

First, we need a smartcard. For this article, I'm using an ePass2000 USB device.

![epass](/assets/epass.png)

I don't necessarily recommend it, as I've not managed to get it to work under non-Windows operating systems. However, it is cute and it works, and the Windows drivers are available online for free (I find this very convenient, as I've managed to misplace the drivers to some of my other smartcard devices...). Most important of all though, is that it supports the PKCS#11 standard. This means the device should be compatible with all other software also supporting the PKCS#11 standard, such as Firefox and QCA.

In CMS Signer, there is a place to perform PKCS#11 configuration.

![cmssigner1](/assets/cmssigner1.png)

In here, we must set the path to the PKCS#11 `.DLL` file (on other platforms this may be a `.so` or `.dylib` file). For the ePass2000, this is `C:\WINDOWS\system32\ep2pk11.dll`. There are also some other low-level options you can set, but generally the defaults should be fine. The PKCS#11 provider plugin for QCA was written by Alon Bar-Lev, and it makes use of his pkcs11-helper library, which is where most of these options come from.

![cmssigner3](/assets/cmssigner3.png)

It is unfortunate that this configuration step is needed at all, but there is currently no known or standard way to locate a user's desired PKCS#11 `.DLL` files. You would have to perform a very similar configuration step in Firefox as well.

Now that the drivers are configured, it should be possible to browse for our smartcard.

![cmssigner4](/assets/cmssigner4.png)

We can then choose a key:

![cmssigner5](/assets/cmssigner5.png)

The key is then imported into a local keyring. It is important to keep in mind that only the public portion of the key is imported. The private portion is not extractable from the smartcard. We can select this key to be used for signing, and enter some data to sign:

![cmssigner6](/assets/cmssigner6.png)

... and hit the Sign button, which prompts for a PIN:

![cmssigner7](/assets/cmssigner7.png)

... and now the CMS-format signature is generated:

![cmssigner8](/assets/cmssigner8.png)

CMS Signer can also perform signature verification, as long as the Text and Signature fields are filled. Since they are filled at this time, we can immediately verify the signature after signing.

![cmssigner9](/assets/cmssigner9.png)

Looks good. :)

Both Psi 0.11 and KDE 4 use QCA. In the future, expect to see smartcard support in these projects.

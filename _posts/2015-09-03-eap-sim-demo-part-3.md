---
layout: post
title: Reading a SIM Card
description: How to read a SIM card to extract EAP-SIM data
tags:
  - eap-sim
  - freeradius
  - wi-fi
  - 802.1x
published: false
---

## Introduction

Welcome to Part 3 of this six part series on building an EAP-SIM demo.

### Prerequisites

See here for prerequisites.

## Install your SIM Card Reader drivers

I'm using an [OmniKey 6121 v2](http://www.hidglobal.com/products/readers/omnikey/6121) reader but any reader should be fine, as long as you have drivers available for your operating system.

The Omnikey 6121 OS X drivers are availble for download [here](http://www.hidglobal.com/drivers/14965) and installation will require a reboot:

<figure>
  <img src="/images/omnikey-installation.png" alt="">
</figure>

## Install Python and Swig.

The tools I am going to use are Python-based, so you need to ensure you have Python installed.  

Although Python ships with OS X, I prefer to use the most up-to-date versions of Python, which you can install using [Homebrew](http://brew.sh).

{% highlight console %}
$ brew install python
==> Installing dependencies for python: readline, sqlite, gdbm
==> Installing python dependency: readline
...
...
==> Summary
🍺  /usr/local/Cellar/python/2.7.10_2: 4866 files, 76M
{% endhighlight %}

We also need to install [Swig](http://swig.org), which is a library that provides bindings from languages such as Python into C/C++ applications.  Swig is required to install pyscard.

 {% highlight console %}
 $ brew install swig
==> Installing swig dependency: pcre
==> Downloading https://homebrew.bintray.com/bottles/pcre-8.37.yosemite.bottle.t
######################################################################## 100.0%
==> Pouring pcre-8.37.yosemite.bottle.tar.gz
🍺  /usr/local/Cellar/pcre/8.37: 146 files, 5.9M
==> Installing swig
==> Downloading https://homebrew.bintray.com/bottles/swig-3.0.7.yosemite.bottle.
######################################################################## 100.0%
==> Pouring swig-3.0.7.yosemite.bottle.tar.gz
🍺  /usr/local/Cellar/swig/3.0.7: 729 files, 7.1M

## Install Pyscard

Pyscard is a Python module that provides smart card support for Python applications.

For my environment, Pyscard 1.6.16 only seems to work - there is a later version of Pyscard available (1.9.0) but this 
The pysim library that we will install later only seems to work



$ pcsctest

MUSCLE PC/SC Lite Test Program

Testing SCardEstablishContext    : Command successful.
Testing SCardGetStatusChange 
Please insert a working reader   : Command successful.
Testing SCardListReaders         : Command successful.
Reader 01: OMNIKEY AG CardMan 6121
Enter the reader number          : 1
Waiting for card insertion         
                                 : Command successful.
Testing SCardConnect             : Command successful.
Testing SCardStatus              : Command successful.
Current Reader Name              : OMNIKEY AG CardMan 6121
Current Reader State             : 0x54
Current Reader Protocol          : 0x0
Current Reader ATR Size          : 21 (0x15)
Current Reader ATR Value         : 3B 9E 96 80 1F C7 80 31 E0 73 FE 21 1B 66 D0 01 77 8B 0D 00 F0 
Testing SCardDisconnect          : Command successful.
Testing SCardReleaseContext      : Command successful.
Testing SCardEstablishContext    : Command successful.
Testing SCardGetStatusChange 
Please insert a working reader   : Command successful.
Testing SCardListReaders         : Command successful.
Reader 01: OMNIKEY AG CardMan 6121
Enter the reader number          : ^C


https://gist.github.com/mixja/c7a876e8c987073a7071
---
layout: post
title: EAP-SIM Overview
description: An overview of how EAP-SIM works
tags:
  - eap-sim
  - freeradius
  - wi-fi
  - 802.1x
published: false
---

## Introduction

Welcome to Part 2 of this six part series on building an EAP-SIM demo.  

In this article, I will introduce the basic concepts of EAP-SIM.  It's important to get the big picture of what's going on and hopefully the background theory you learn here will help the latter series of posts in this series make more sense. 

### What is EAP-SIM?

EAP-SIM is an *extensible authentication protocol* (EAP) based protocol that leverages existing SIM based authentication methods to allow mobile Wi-Fi devices with SIM cards to mutually authenticate with a Wi-Fi network.

EAP-SIM is obviously the domain of mobile network operators - 

## EAP-SIM Components


  
See here for prerequisites.

## Install your SIM Card Reader drivers

I'm using an [OmniKey 6121 v2](http://www.hidglobal.com/products/readers/omnikey/6121) reader but any reader should be fine, as long as you have drivers available for your operating system.

The Omnikey 6121 OS X drivers are availble for download [here](http://www.hidglobal.com/drivers/14965) and installation will require a reboot:

<figure>
  <img src="/images/omnikey-installation.png" alt="">
</figure>

## Verify your SIM Card Reader

At this point, you should verify your SIM card reader is working and recognised by your system.

On OS X, this is easily achieved via the system utility `pcsctest`:

{% highlight console %}
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
{% endhighlight %}

Note when executing the `pscstest` command, at the `Enter the reader number:` prompt you need to enter `1`. 

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
...
...
==> Pouring swig-3.0.7.yosemite.bottle.tar.gz
🍺  /usr/local/Cellar/swig/3.0.7: 729 files, 7.1M

## Install Pyscard

Pyscard is a Python module that provides smart card support for Python applications.

For my environment, Pyscard **1.6.16** only seems to work - there is a later version of Pyscard available (1.9.0) but the Pysim library that we will install later only seems to work with version 1.6.16.

You can download [Pyscard 1.16.16 here](http://sourceforge.net/projects/pyscard/files/pyscard/pyscard%201.6.16/pyscard-1.6.16.tar.gz/download).

Once you have downloaded Pyscard, extract the Pyscard source and install it as follows:

{% highlight console %}
$ tar zxvf pyscard-1.6.16.tar.gz
...
...
x pyscard-1.6.16/LICENSE
x pyscard-1.6.16/setup.py
x pyscard-1.6.16/README
$ cd pyscard-1.6.16
pyscard-1.6.16 $ python setup.py install
running install
running build
running build_py
running build_ext
running install_lib
running install_egg_info
Removing /usr/local/lib/python2.7/site-packages/pyscard-1.6.16-py2.7.egg-info
Writing /usr/local/lib/python2.7/site-packages/pyscard-1.6.16-py2.7.egg-info
{% endhighlight %}

## Install Pysim 

[Pysim](http://cgit.osmocom.org/pysim/) is a SIM card management tool that allows you to read your SIM card.

To install Pysim, we need to clone the Pysim GIT repository:

{% highlight console %}
$ git clone git://git.osmocom.org/pysim
Cloning into 'pysim'...
remote: Counting objects: 229, done.
remote: Compressing objects: 100% (226/226), done.
remote: Total 229 (delta 148), reused 0 (delta 0)
Receiving objects: 100% (229/229), 61.30 KiB | 41.00 KiB/s, done.
Resolving deltas: 100% (148/148), done.
Checking connectivity... done.
$ cd pysim
pysim $ 
{% endhighlight %}

We also need to add a separate Python application called `pySim-run-gsm.py` to the root of the pysim folder you just cloned.

`pySim-run-gsm.py` is the application we will use to read our SIM card.  Note I have also published a copy of this at https://gist.github.com/mixja/c7a876e8c987073a7071.

{% highlight python %}
#!/usr/bin/env python

#
# Utility to run the 2G gsm algorithm on the SIM card
# used to generate authentication triplets for EAP-SIM
#
# Copyright (C) 2009  Sylvain Munaut <tnt@246tNt.com>
# Copyright (C) 2010  Harald Welte <laforge@gnumonks.org>
# Copyright (C) 2013  Alexander Chemeris <alexander.chemeris@gmail.com>
# Copyright (C) 2010  Harald Welte <laforge@gnumonks.org>
# Copyright (C) 2013  Alexander Chemeris <alexander.chemeris@gmail.com>
# Copyright (C) 2013  Darell Tan <darell.tan@gmail.com>
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
#

import hashlib
from optparse import OptionParser
import os
import random
import time
import re
import sys

try:
	import json
except ImportError:
	# Python < 2.5
	import simplejson as json

from pySim.commands import SimCardCommands
from pySim.utils import h2b, swap_nibbles, rpad, dec_imsi, dec_iccid


def parse_options():

	parser = OptionParser(usage="usage: %prog [options]")

	parser.add_option("-d", "--device", dest="device", metavar="DEV",
			help="Serial Device for SIM access [default: %default]",
			default="/dev/ttyUSB0",
		)
	parser.add_option("-b", "--baud", dest="baudrate", type="int", metavar="BAUD",
			help="Baudrate used for SIM access [default: %default]",
			default=9600,
		)
	parser.add_option("-p", "--pcsc-device", dest="pcsc_dev", type='int', metavar="PCSC",
			help="Which PC/SC reader number for SIM access",
			default=None,
		)
	parser.add_option("-n", "--iterations", dest="iterations", type='int', metavar="NUM",
			help="Number of iterations to run the GSM algorithm",
			default=100,
		)

	(options, args) = parser.parse_args()

	if args:
		parser.error("Extraneous arguments")

	return options


if __name__ == '__main__':

	# Parse options
	opts = parse_options()

	# Connect to the card
	if opts.pcsc_dev is None:
		from pySim.transport.serial import SerialSimLink
		sl = SerialSimLink(device=opts.device, baudrate=opts.baudrate)
	else:
		from pySim.transport.pcsc import PcscSimLink
		sl = PcscSimLink(opts.pcsc_dev)

	# Create command layer
	scc = SimCardCommands(transport=sl)

	# Wait for SIM card
	sl.wait_for_card()

	# Program the card
	print("Reading ...")

	# EF.IMSI
	(res, sw) = scc.read_binary(['3f00', '7f20', '6f07'])
	if sw == '9000':
		print("IMSI: %s" % (dec_imsi(res),))
	else:
		print("IMSI: Can't read, response code = %s" % (sw,))
	imsi = dec_imsi(res)

	# run the algorithm here and output results
	print('%-16s %-32s %-8s %s' % ('# IMSI', 'RAND', 'SRES', 'Kc'))
	for i in xrange(opts.iterations):
		rand = ''.join('%02x' % ord(x) for x in os.urandom(16))
		(res, sw) = scc.run_gsm(rand)
		if sw == '9000':
			SRES, Kc = res[:8], res[8:]
			print('%s,%s,%s,%s' % (imsi, rand, SRES, Kc))
			if i % 5 == 0: time.sleep(2)
		else:
			print('cannot run gsm algo. response code = %s' % (sw,))
			break
{% endhighlight %}

## Reading your SIM Card

We are now ready to read our SIM card.  

For EAP-SIM authentication, we need three sets of *triplets* - a triplet is the combination of the following (see xxxx part 1 for a detailed discussion on which of these values is):

- RAND (Random Number)   
- SRES (Signed Response)   
- Kc (Cipher Key)

We can obtain these triplets by running the `pySim-run-gsm.py` application from the `pysim` source folder as shown below.  

After the application generates at least three triplets, use `CTRL+C` to exit the application. 

{% highlight console %}
pysim $ ./pySim-run-gsm.py -p 0
Reading ...
IMSI: 530052190977328
# IMSI           RAND                             SRES     Kc
530052190977328,8bad06de2921dd5cbf0e322ac310d696,488977b0,3aedf9356020e1bb
530052190977328,59c0bc6dae909aca732ff44914d845e2,2865bd5e,24e24e202ab841c1
530052190977328,17528716ee72626c5cb85ac167389a4e,9ee3a926,4c1b2540b7149e0f
...
...
...
^C
{% endhighlight %}

## Next steps

We've reached the end of this article, and it's important that you record the following from the output of the previous section:

- IMSI  
- Three sets of RAND/SRES/Kc triplet values

We'll be using these values later on in Part 4 when we configure FreeRADIUS for EAP-SIM authentication.
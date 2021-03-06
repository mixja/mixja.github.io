---
published: false
---

## Building an EAP-SIM Demo - Introduction

Welcome to this 6 part series on building an EAP-SIM demonstration lab.

In this series, I'll demonstrate how to build a simple lab that will enable you to test and demonstrate authenticating to a Wi-Fi network using EAP-SIM using real wireless infrastructure and mobile phones.

This introductory article will introduce the environment we will build and describe the prerequisites you will need if you want to build your own demonstration lab.

### Series Overview

Here's a list of each of the articles in this series:

1. Introduction (this article)
2. EAP-SIM Authentication Overview
3. Reading your SIM Card
4. Building the Wireless Infrastructure
5. Configuring the RADIUS Server
6. Configuring an iPhone for EAP-SIM

### Environment

A picture is worth a thousand words so lets get straight to it - here's a high level view of what we are going to build.




### Prerequisites

You are going to need a few pieces of hardware and software for this lab.  

- **Mobile handset with Wi-Fi and 802.1x/EAP-SIM support** - I'll use an iPhone 6 in this demonstration, but other iPhones with a recent iOS version should work OK. Android 4.x and above also supports EAP-SIM.  For iPhones you will also need the [Apple Configurator](https://itunes.apple.com/en/app/apple-configurator/id434433123?mt=12) to help configure your iPhone for EAP-SIM.
- **SIM Card Reader** - in this demonstration I'll use an [OmniKey 6121 v2](http://www.hidglobal.com/products/readers/omnikey/6121) reader but any reader with drivers for your operating system should work OK.
- **Wireless Access Point** - in this demonstration I'll use a Cisco Aironet 3502 access point that is paired with a Cisco Virtual Wireless LAN Controller (vWLC), but any access point that supports 802.1x should be able to work.
- **Cisco vWLC** - if you are a registered Cisco user you should be able to [download this from Cisco here](https://software.cisco.com/download/release.html?mdfid=284464214&softwareid=280926587&release=8.1.111.0).  If you don't have a Cisco account or your account does not have privileges, Google is your friend.  I'll be running this on VMWare Fusion 7.1.2 (OS X) in this demonstration.
- **Docker/Vagrant** - I'll use these tools to bootstrap prebuilt images I've created for FreeRADIUS 3.x.  Images are available for Docker and Vagrant.
- **OS X** - I'll be using OS X Yosemite 10.10.5 exclusively to drive everything in this demonstration, but there's no reason you can't perform almost all of this lab on another operating system.








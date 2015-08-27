---
layout: post
title: DPDK on an Intel NUC
description: How to build DPDK and Seastar on an Intel NUC
tags: 
  - dpdk
  - seastar
  - networking
published: true
---


## Introduction

The DPDK (Data Plane Development Kit) is an exciting project that delivers high performance networking by allowing low-level networking functions to be performed in user space applications, instead of in the Linux kernel.  

If you are not familiar with DPDK, you can find out more information at the following links:

- [http://dpdk.org](http://dpdk.org)
- [http://http://dpdk.readthedocs.org/en/latest/index.html](http://dpdk.readthedocs.org/en/latest/index.html)

In this article, I will show you how to build and configure DPDK and run a simple web application using the Seastar application framework.

## Seastar
[Seastar](http://www.seastar-project.org) is a very young [open source project](https://github.com/cloudius-systems/seastar) created by Cloudius Systems, the company behind [OSv](http://osv.io).  

As described on the project home page:

> Seastar is an advanced, open-source C++ framework for high-performance server applications on modern hardware.

One of the fascinating features of Seastar is its support of DPDK on Linux systems.  This is one of the first application frameworks I have seen that supports DPDK so I was very excited when I stumbled upon it.

I'm currently working with [Vert.x](http://vertx.io), which is an excellent application toolkit for building high performance reactive applications on the JVM.  Vert.x is well known for its excellent performance, and I've struggled to find alternatives that match it in terms of speed.

On paper, Seastar looks like it could be a worthy challenger to Vert.x in terms of performance, given it is developed in C++ and its support of DPDK.  

So I decided to give Seastar a go.

## Hardware
The hardware used was an [Intel NUC](http://www.intel.com/content/www/us/en/nuc/nuc-kit-d34010wyk.html) with the following specifications:

- Intel Core i3 4010U (dual-core 1.7GHz)
- 4GB RAM
- 30GB SSD
- Intel I218 Gigabit Ethernet NIC
- Hyperthreading disabled (this is enabled by default in the UEFI BIOS)

DPDK only works on [supported network interfaces](http://dpdk.org/doc/nics) and you'll note that the I218 is not on the supported list.  Unperturbed I decided to press on and as you'll find out later on, I actually had to modify the DPDK source code to get DPDK working with the I218 network interface.

## Preparing the Operating System
Most of the examples in the Seastar [documentation](https://github.com/cloudius-systems/seastar) are based upon Fedora so I decided to use Fedora 22 Server for my testing.  I'll skip the details on installation of Fedora 22 - I just used a typical installation and didn't do anything special during the installation.

Both Seastar and DPDK need to be built from source, so you need to install a number of development related dependencies.

When building DPDK, you need to create a kernel module so it's important to install the relevent kernel development packages that is matched to your system kernel.  This can be achieved most quickly by first upgrading your system and then installing the kernel development packages:  

{% highlight console %}
dnf upgrade -y
dnf install kernel-devel.x86_64 -y
{% endhighlight %}

If you miss the upgrade step, chances are your system kernel will be an older kernel version than the kernel development packages and things will break.

Seastar has a number of development dependencies, which are listed in the Fedora 21 section of the main README file on the Seastar Github home project page:

{% highlight console %}
dnf install gcc-c++ libaio-devel ninja-build ragel hwloc-devel numactl-devel \
                    libpciaccess-devel cryptopp-devel xen-devel boost-devel \
                    libxml2-devel -y
{% endhighlight %}

In addition to the above, you need Git (to download the Seastar and DPDK source) and a couple of other libraries that are required when building Seastar on Fedora 22:

{% highlight console %}
dnf install git libubsan libasan -y
{% endhighlight %}

### Configuring Hugepages

Hugepage support is required for the large memory pool allocation used for packet buffers.  I won't go into details but you can read more about it in this [document](http://dpdk.readthedocs.org/en/latest/linux_gsg/sys_reqs.html). 

There are two typical hugepage sizes:

- 2MB
- 1GB

All modern hardware should at least support 2MB hugepages but you can check if your system supports 1GB hugepages as follows:

{% highlight console %}
# cat /proc/cpuinfo
processor	: 0
vendor_id	: GenuineIntel
cpu family	: 6
model		: 69
model name	: Intel(R) Core(TM) i3-4010U CPU @ 1.70GHz
stepping	: 1
microcode	: 0x1c
cpu MHz		: 900.003
cache size	: 3072 KB
physical id	: 0
siblings	: 2
core id		: 0
cpu cores	: 2
apicid		: 0
initial apicid	: 0
fpu		: yes
fpu_exception	: yes
cpuid level	: 13
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat 
                  pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx
                  pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl 
                  xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor 
                  ds_cpl vmx est tm2 ssse3 fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic 
                  movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 
                  ida arat epb pln pts dtherm tpr_shadow vnmi flexpriority ept vpid 
                  fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid xsaveopt
bugs		:
bogomips	: 3391.95
clflush size	: 64
cache_alignment	: 64
address sizes	: 39 bits physical, 48 bits virtual
power management:
...
...
{% endhighlight %}

You are looking for a couple of values in the flags section:

- `pse` - 2MB hugepages are supported
- `pdpe1gb` - 1GB hugepages are supported

In the example above, you can see that 1GB hugepages are supported.

To enable hugepages you need to first add the following line to `/etc/default/grub`: 

**2MB Hugepages**
{% highlight console %}
GRUB_CMDLINE_LINUX_DEFAULT="default_hugepagesz=2M hugepagesz=2M hugepages=512"
{% endhighlight %}

**1GB Hugepages**
{% highlight console %}
GRUB_CMDLINE_LINUX_DEFAULT="default_hugepagesz=1G hugepagesz=1G hugepages=1"
{% endhighlight %}

Both examples above allocate 1GB of memory to hugepages.  In my case, I allocated a single 1GB hugepage.

Next you need to update your Grub configuration.  Note these instructions are for Fedora.

**BIOS systems**
{% highlight console %}
grub2-mkconfig -o /boot/grub2/grub.cfg
{% endhighlight %}

**UEFI systems**
{% highlight console %}
grub2-mkconfig -o /etc/grub2-efi.cfg
{% endhighlight %}

In my case, the Intel NUC is a UEFI system so I used the latter command above.

You now need to mount the hugetable file system by adding the following entry to `/etc/fstab`:

**2MB Hugepages**
{% highlight console %}
hugetlbfs    /dev/hugepages    hugetlbfs    defaults 0 0
{% endhighlight %}

**1GB Hugepages**
{% highlight console %}
hugetlbfs    /dev/hugepages    hugetlbfs    pagesize=1G 0 0
{% endhighlight %}

And finally reboot and verify hugepages has been configured correctly:

{% highlight console %}
# cat /proc/meminfo
MemTotal:        3974272 kB
MemFree:         2549296 kB
MemAvailable:    2749600 kB
...
...
HugePages_Total:       1
HugePages_Free:        1
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:    1048576 kB
DirectMap4k:       86460 kB
DirectMap2M:     1937408 kB
DirectMap1G:     2097152 kB
{% endhighlight %}

### Getting the DPDK and Seastar Source

The Seastar Git repository includes the DPDK repository as a submodule, and under normal circumstances you can build Seastar with DPDK support, without needing to build DPDK manually yourself.   

In my case however, in order to support the Intel I218 network interface card I actually had to make some modifications to the DPDK source code.  The modifications are included in a [fork I created](http://github.com/mixja/dpdk) from the DPDK repository.  

I can't take any credit for the changes - I found a [DPDK Mailing List thread](http://patchwork.dpdk.org/ml/archives/dev/2015-January/010714.html) from January 2015 that discussed a patch for adding I217/I218 support (but looks like it never made it in to the DPDK source) so I took the code from there with a few minor modications.

So for this example you need to clone my forked DPDK repository, and at the same time you should also clone the Seastar repository:

{% highlight console %}
git clone http://github.com/mixja/dpdk
git clone https://github.com/cloudius-systems/seastar.git
{% endhighlight %}

## Building DPDK

At this point, you need to down the network interface that is to be used by DPDK.  

This will allow DPDK to take control of the NIC, outside of the Linux kernel:

{% highlight console %}
[root@localhost dpdk]# ifdown eno1
Device 'eno1' successfully disconnected.
{% endhighlight %}

Next you need to configure your DPDK build options - this is performed by editing the `config/common_linuxapp` file and modifying the following entries:

{% highlight console %}
CONFIG_RTE_LIBTRE_KNI=n
CONFIG_RTE_MBUF_REFCNT_ATOMIC=n
CONFIG_RTE_MAX_MEMSEG=4096
{% endhighlight %}

The configuration settings above are based upon instructions included in the Seastar project documentation.

Finally you need to build the DPDK target environment - this is made very easy by using the `tools/setup.sh` script included in the DPDK source.  

{% highlight console %}
[root@localhost dpdk]# tools/setup.sh

------------------------------------------------------------------------------
 RTE_SDK exported as /root/dpdk
------------------------------------------------------------------------------
----------------------------------------------------------
 Step 1: Select the DPDK environment to build
----------------------------------------------------------
[1] i686-native-linuxapp-gcc
[2] i686-native-linuxapp-icc
[3] ppc_64-power8-linuxapp-gcc
[4] tile-tilegx-linuxapp-gcc
[5] x86_64-ivshmem-linuxapp-gcc
[6] x86_64-ivshmem-linuxapp-icc
[7] x86_64-native-bsdapp-clang
[8] x86_64-native-bsdapp-gcc
[9] x86_64-native-linuxapp-clang
[10] x86_64-native-linuxapp-gcc
[11] x86_64-native-linuxapp-icc
[12] x86_x32-native-linuxapp-gcc

----------------------------------------------------------
 Step 2: Setup linuxapp environment
----------------------------------------------------------
[13] Insert IGB UIO module
[14] Insert VFIO module
[15] Insert KNI module
[16] Setup hugepage mappings for non-NUMA systems
[17] Setup hugepage mappings for NUMA systems
[18] Display current Ethernet device settings
[19] Bind Ethernet device to IGB UIO module
[20] Bind Ethernet device to VFIO module
[21] Setup VFIO permissions

----------------------------------------------------------
 Step 3: Run test application for linuxapp environment
----------------------------------------------------------
[22] Run test application ($RTE_TARGET/app/test)
[23] Run testpmd application in interactive mode ($RTE_TARGET/app/testpmd)

----------------------------------------------------------
 Step 4: Other tools
----------------------------------------------------------
[24] List hugepage info from /proc/meminfo

----------------------------------------------------------
 Step 5: Uninstall and system cleanup
----------------------------------------------------------
[25] Uninstall all targets
[26] Unbind NICs from IGB UIO or VFIO driver
[27] Remove IGB UIO module
[28] Remove VFIO module
[29] Remove KNI module
[30] Remove hugepage mappings

[31] Exit Script
{% endhighlight %}

In the setup.sh script you need to perform the following steps:

1. Option [10] - builds a Linux 64-bit target using GCC.  The target will be created in a directory called `x86_64-native-linuxapp-gcc` in the DPDK source root.
2. Option [13] - loads the IGB_UIO kernel module
3. Option [19] - allows you to bind your network interface to DPDK

The example below demonstrates the final step of binding a network interface to DPDK.  

Note that you need to enter the PCI address of the network interface card (**00:19.0** in this example):

{% highlight console %}
Network devices using DPDK-compatible driver
============================================
<none>

Network devices using kernel driver
===================================
0000:00:19.0 'Ethernet Connection I218-V' if=eno1 drv=e1000e unused=igb_uio

Other network devices
=====================
<none>

Enter PCI address of device to bind to IGB UIO driver: 00:19.0
OK

Press enter to continue ...
{% endhighlight %}

You can use option [19] again to verify the network interface is now bound to DPDK:

{% highlight console %}
Network devices using DPDK-compatible driver
============================================
0000:00:19.0 'Ethernet Connection I218-V' if=eno1 drv=e1000e unused=igb_uio

Network devices using kernel driver
===================================
<none>

Other network devices
=====================
<none>

Enter PCI address of device to bind to IGB UIO driver: 
{% endhighlight %}

Once you are done, choose option [31] to exit the setup script.

## Building Seastar
You are now ready to configure and build Seastar.  

As described earlier, because of the requirement to use patched DPDK source code, you need to configure Seastar to build against a precompiled DPDK package (which you created in the last section).  

You can do this by including the `--dpdk-target <path to dpdk source>/x86_64-native-linuxapp-gcc` flag when configuring the Seastar build:

{% highlight console %}
seastar# ./configure.py --dpdk-target /root/dpdk/x86_64-native-linuxapp-gcc \
                        --disable-xen \
                        --with apps/httpd/httpd
{% endhighlight %}

The `configure.py` script is located in the Seastar source root - note in addition to the `--dpdk-target` flag the following flags are required:

- `--disable-xen` - this is required as DPDK and Xen support are mutually exclusive
- `--with` - this allows you to constrain the sample applications that will be built, reducing the number of libraries that need to built to just those required to support the specified application.  For this example, you only need the HTTP sample application.

With the configuration in place, you can now build Seastar:

{% highlight console %}
seastar# ninja-build
...
[81/82] CXX build/release/http/transformers.o
[82/82] LINK build/release/apps/httpd/httpd
seastar# 
{% endhighlight %}

The build will take a few minutes - you can see above 82 libraries are built.

If you omitted the `--with` flag in the configuration (meaning build everything) you would need to build 148 libraries.

## Running Seastar
At this point, you are ready to run Seastar.  Seastar includes a basic example web application, which you can run as demonstrated below:

{% highlight console %}
seastar# build/release/apps/httpd/httpd \
           --network-stack native \
           --dpdk-pmd \ 
           --dhcp 0 \ 
           --host-ipv4-addr 192.168.1.200 \ 
           --netmask-ipv4-addr 255.255.255.0 \ 
           --collectd 0 \ 
           --smp 2 \ 
           --port 10000
{% endhighlight %}

There's quite a few options above you need to specify.  Note that you can use the `httpd --help` command to review all of the available options.  Each of the  flags used in the example are described below:
which are described below:

- `--network-stack native` - this configures the application to run using the Seastar native network stack, which is required when using DPDK.
- `--dpdk-pmd` - this configures the application to use DPDK
- `--dhcp 0` - this disables DHCP
- `--host-ipv4-addr` - specifies the IPv4 address to run the application on
- `--netmask-ipv4-addr` - specifies the subnet mask of the configured IPv4 address
- `--collectd 0` - disables the CollectD daemon, which is used for collecting metrics and statistics
- `--smp 2` - this specifies the number of threads.  For maximum performance, this should equal the number of physical CPU cores.
- `--port 10000` - the port the web server will run on.  If omitted, the server will run on port 10000.

The application takes a few seconds to start up as shown in the output below:

{% highlight console %}
seastar# build/release/apps/httpd/httpd \
           --network-stack native \
           --dpdk-pmd \ 
           --dhcp 0 \ 
           --host-ipv4-addr 192.168.1.200 \ 
           --netmask-ipv4-addr 255.255.255.0 \ 
           --collectd 0 \ 
           --smp 2 \ 
           --port 10000
EAL: Detected lcore 0 as core 0 on socket 0
EAL: Detected lcore 1 as core 1 on socket 0
EAL: Support maximum 128 logical core(s) by configuration.
EAL: Detected 2 lcore(s)
EAL: VFIO modules not all loaded, skip VFIO support...
EAL: Setting up physically contiguous memory...
EAL: Ask a virtual area of 0x40000000 bytes
EAL: Virtual area found at 0x7fe940000000 (size = 0x40000000)
EAL: Requesting 1 pages of size 1024MB from socket 0
EAL: TSC frequency is ~1696073 KHz
EAL: Master lcore 0 is ready (tid=348c7900;cpuset=[0])
EAL: lcore 1 is ready (tid=30a99700;cpuset=[1])
EAL: PCI device 0000:00:19.0 on NUMA socket -1
EAL:   probe driver: 8086:1559 rte_em_pmd
EAL:   PCI memory mapped at 0x7fe980000000
EAL:   PCI memory mapped at 0x7fe980020000
PMD: eth_em_dev_init(): port_id 0 vendorID=0x8086 deviceID=0x1559
ports number: 1
Port 0: max_rx_queues 1 max_tx_queues 1
Port 0: using 1 queue
LRO is off
Port 0 init ... done:
Creating Tx mbuf pool 'dpdk_net_pktmbuf_pool0_tx' [1024 mbufs] ...
Creating Rx mbuf pool 'dpdk_net_pktmbuf_pool0_rx' [1024 mbufs] ...
PMD: eth_em_rx_queue_setup(): sw_ring=0x7fe97f590ac0 hw_ring=0x7fe97f591bc0 dma_addr=0xbf591bc0
PMD: eth_em_tx_queue_setup(): sw_ring=0x7fe97f57e980 hw_ring=0x7fe97f580a80 dma_addr=0xbf580a80
PMD: eth_em_flow_ctrl_set(): Rx packet buffer size = 0x6800
Port 0: Enabling HW FC
PMD: eth_em_start(): <<

Checking link status
Created DPDK device
.done
Port 0 Link Up - speed 1000 Mbps - full-duplex
Seastar HTTP server listening on port 10000 ...
{% endhighlight %}

## Testing Seastar
The sample HTTP application is now up and running.  Running a basic test should demonstrate the application is working:

{% highlight console %}
$ curl 192.168.1.200:10000
"hello"
{% endhighlight %}

And a quick performance test should show you some pretty impressive numbers:

{% highlight console %}
$ weighttp -n 1000000 -c 100 -k 192.168.1.200:10000
weighttp - a lightweight and simple webserver benchmarking tool

starting benchmark...
spawning thread #1: 100 concurrent requests, 1000000 total requests
progress:  10% done
progress:  20% done
progress:  30% done
progress:  40% done
progress:  50% done
progress:  60% done
progress:  70% done
progress:  80% done
progress:  90% done
progress: 100% done

finished in 6 sec, 706 millisec and 434 microsec, 149110 req/s, 18347 kbyte/s
requests: 1000000 total, 1000000 started, 1000000 done, 1000000 succeeded, 0 failed, 0 errored
status codes: 1000000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 126000000 bytes total, 119000000 bytes http, 7000000 bytes data
{% endhighlight %}

**150,000 requests per second - not too bad for $300US!**

In an upcoming post, I'll publish some performance stats comparing Node.js, Vert.x, Seastar and Seastar with DPDK.  

I'm sure you already know who the winner will be ;)

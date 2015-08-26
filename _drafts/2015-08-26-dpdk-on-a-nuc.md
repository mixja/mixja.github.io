---
published: false
---

## DPDK on a NUC

The DPDK (Data-plane Development Kit) is an exciting project that delivers staggering performance from commodity off the shelf (COTS) hardware and ...

In this article, I will show you how to build and configure DPDK and run a simple web application using the Seastar application framework.

### Seastar
[Seastar](http://www.seastar-project.org) is a very young open source project created by Cloudius Systems, the company behind [OSv](http://osv.io).  As described on the project home page:

> Seastar is an advanced, open-source C++ framework for high-performance server applications on modern hardware.

One of the fascinating features of Seastar is its support of DPDK on Linux systems.  This is one of the first application frameworks I have seen that supports DPDK so I was very excited when I stumbled upon it.

I'm currently working with [Vert.x](http://vertx.io), which is an excellent application framework toolset for building high performance reactive applications on the JVM.  Vert.x is well known for its lofty performance, and I've struggled to find alternatives that match it for performance.  

On paper, Seastar looks like it could be a worthy challenger to Vert.x in terms of performance, given it is developed in C++ and its support of DPDK.  So I decided to give Seastar a go.

### Hardware


### Preparing the Operating System
Most of the examples in the Seastar [documentation](https://github.com/cloudius-systems/seastar) are based upon Fedora so I decided to use Fedora 22 Server for my testing.  I'll skip the details on installation of Fedora 22 - I just used a typical installation and didn't do anything special during the installation.

Both Seastar and DPDK need to be built from source, so you need to install a number of development related dependencies.

When building DPDK, you need to create a kernel module so it's important to install the relevent kernel development packages that is matched to your system kernel.  This can be achieved most quickly by first upgrading your system and then installing the kernel development packages:  

```console
dnf upgrade -y
dnf install kernel-devel.x86_64 -y
```

If you miss the upgrade step, chances are your system kernel will be an older kernel version than the kernel development packages and things will break.

Seastar has a number of development dependencies, which are listed in the Fedora 21 section of the main README file on the Seastar Github home project page:

```console
dnf install gcc-c++ libaio-devel ninja-build ragel hwloc-devel numactl-devel libpciaccess-devel cryptopp-devel xen-devel boost-devel libxml2-devel
```

In addition to the above, I installed Git (to download the Seastar and DPDK source) and a couple of other libraries that are required when building Seastar on Fedora 22:

```console
dnf install git libubsan libasan -y
```
### Building DPDK

We need to build DPDK first as to support DPDK, Seastar must be built with DPDK support enabled.  Although the Seastar Git repository includes the DPDK repository as a submodule, to support the Intel NUC I218 NIC card I actually had to make some source code modifications.  

These modules are included in a [fork I created](http://github.com/mixja/dpdk) from the DPDK repository.  I can't take any credit for the changes - I found a [DPDK Mailing List thread](http://patchwork.dpdk.org/ml/archives/dev/2015-January/010714.html) from January 2015 that discussed a patch for adding I217/I218 support (but looks like it never made it in to the DPDK source) so I took the code from there with a few minor modications.

So first of all we clone my forked DPDK repository, and we'll also clone the Seastar repository as we are soon going to lose our network connectivity:

```console
git clone http://github.com/mixja/dpdk
git clone https://github.com/cloudius-systems/seastar.git
```

At this point, we need to down the network interface that is to be used by DPDK.  This will allow DPDK to take control of the NIC, outside of the Linux kernel.  

Next we need to build the DPDK target environment - this is made very easy by using the `tools/setup.sh` script in the DPDK source.  Choose option [10], which builds a Linux 64-bit target using GCC.  

After the target environment is built, you need to insert the IGB_UIO kernel module by choosing option [12] and then create 



Building Seastar is very straight forward and I followed the same steps 




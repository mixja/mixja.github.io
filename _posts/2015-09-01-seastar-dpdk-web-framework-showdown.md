---
layout: post
title: Seastar DPDK Web Framework Showdown
description: Seastar DPDK takes on Node.js, Go and Vert.x.
tags:
  - dpdk
  - seastar
  - networking
  - intel
  - nuc
  - go
  - gin
  - vert.x
  - node.js
  - sdn
  - nfv
image:
  feature: dragons-bg.jpg
  credit: Keith Cuddeback
  creditlink: https://www.flickr.com/photos/in2photos/7198977392/
published: true
---

[In my last post](http://pseudo.co.de/dpdk-on-an-intel-nuc/), I introduced Seastar and DPDK and how to run Seastar with DPDK on an Intel NUC.

In this post, I'm going to benchmark the performance of Seastar/DPDK with a number of other web frameworks.  Here is the complete list of frameworks that will be benchmarked:

- [Node.js](http://nodejs.org)
- [Go](http://golang.org) (using [Gin web framework](https://github.com/gin-gonic/gin))
- [Vert.x](http://vertx.io)
- [Seastar (using Linux network stack)](https://github.com/cloudius-systems/seastar)
- [Seastar (using DPDK network stack)](https://github.com/cloudius-systems/seastar/blob/master/README-DPDK.md)

## Environment

For each benchmark, the web framework will run on an Intel NUC with the following specifications:

- Intel Core i3 4010U (dual-core 1.7GHz)
- 4GB RAM
- 30GB mSATA SSD
- Intel I218 Gigabit Ethernet NIC
- Hyperthreading disabled
- Fedora 22 running kernel 4.1.6(200)

Performance is measured using the `weighttp` test client running on a current generation Mac Pro with the following specification:

- Intel Xeon E5 (quad-core 3.7GHz)
- 64GB RAM
- 512GB PCIe SSD
- OS X Yosemite 10.10.4
- weighttp version 0.3

## Testing Methodology

The testing methodology is very simple:

- The web application returns the text "hello" with a 200 OK status code in response to an HTTP request from the client
- For each test scenario, five separate measurements are collected
- The minimum and maximum measurements are discarded with the average of the remaining measurements used as the final result

For all test scenarios, the following conditions are applied:

- 1M Requests (this was reduced to 250,000 requests for Node.js)
- 4 threads
- HTTP Keepalives enabled
- A varying range of concurrent connections (50, 100, 500 and 1000) are tested

The following example demonstrates using `weighttp` with the options specified above to collect a measurement:

{% highlight console %}
$ weighttp -n 1000000 -c 500 -t 4 -k 192.168.1.200:10000
weighttp - a lightweight and simple webserver benchmarking tool

starting benchmark...
spawning thread #1: 125 concurrent requests, 250000 total requests
spawning thread #2: 125 concurrent requests, 250000 total requests
spawning thread #3: 125 concurrent requests, 250000 total requests
spawning thread #4: 125 concurrent requests, 250000 total requests
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

finished in 5 sec, 208 millisec and 134 microsec, 192007 req/s, 23625 kbyte/s
requests: 1000000 total, 1000000 started, 1000000 done, 1000000 succeeded, 0 failed, 0 errored
status codes: 1000000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 126000000 bytes total, 119000000 bytes http, 7000000 bytes data
{% endhighlight %}

## Preparing the Environment

### Node.js

Node.js is installed using the `dnf install nodejs` command:

{% highlight console %}
[root@localhost ~]# dnf install nodejs -y
...
...
[root@localhost ~]# node --version
v0.10.36
{% endhighlight %}

The following application code in a single file `app.js` will be used:

{% highlight javascript %}
var cluster = require('cluster');
var http = require('http');
var numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  // Fork workers.
  for (var i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', function(worker, code, signal) {
    console.log('worker ' + worker.process.pid + ' died');
  });
} else {
  // Workers can share any TCP connection
  // In this case its a HTTP server
  http.createServer(function(req, res) {
    res.writeHead(200);
    res.end("hello");
  }).listen(8000);
}
{% endhighlight %}

And to run the application:

{% highlight console %}
[root@localhost ~]# node app.js
{% endhighlight %}

### Go

Go is installed using the `dnf install golang` command, after which we need to set the GOPATH environment variable:

{% highlight console %}
[root@localhost ~]# dnf install golang -y
...
...
[root@localhost ~]# go version
go version go1.4.2 linux/amd64
[root@localhost ~]# mkdir -p /root/go
[root@localhost ~]# export GOPATH=/root/go
{% endhighlight %}

We will be using the [Gin web framework](https://github.com/gin-gonic/gin) and to install the required source on our system we use the `go get` command:

{% highlight console %}
[root@localhost ~]# go get github.com/gin-gonic/gin
{% endhighlight %}

The following application code in a single file `app.go` will be used:

{% highlight go %}
package main

import "runtime"
import "github.com/gin-gonic/gin"

func main() {
    // r := gin.Default()
    runtime.GOMAXPROCS(runtime.NumCPU())
    gin.SetMode(gin.ReleaseMode)
    r := gin.New()

    r.GET("/", func(c *gin.Context) {
        c.String(200, "hello")
    })
    r.Run(":8000") // listen and serve on 0.0.0.0:8000
}
{% endhighlight %}

And to run the application:

{% highlight console %}
[root@localhost ~]# go run app.go
{% endhighlight %}

### Vert.x

Vert.x runs on the JVM so requires you to first install Java and then download and install the Vert.x runtime.  The Vert.x runtime requires the JDK, so ensure you install this rather than the JRE. 

> For production Vert.x applications, you normally would package your application as a JAR and simply run the application directly from the JVM.  To facilitate simple testing, I am using the Vert.x runtime which allows you to run source files directly.    

{% highlight console %}
[root@localhost ~]# dnf install java-1.8.0-openjdk-devel -y
...
...
[root@localhost ~]# wget https://bintray.com/artifact/download/vertx/downloads/vert.x-3.0.0-full.tar.gz
...
...
[root@localhost ~]# tar zxvf vert.x-3.0.0-full.tar.gz
vert.x-3.0.0/lib/vertx-core-3.0.0.jar
vert.x-3.0.0/lib/netty-common-4.0.28.Final.jar
...
...
[root@localhost ~]# vert.x-3.0.0/bin/vertx -version
3.0.0
{% endhighlight %}

The following application code in a single file `WebService.java` will be used:
{% highlight java %}
import io.vertx.core.AbstractVerticle;

public class WebService extends AbstractVerticle {
    @Override
    public void start() throws Exception {
        // Create HTTP Server
        vertx.createHttpServer().requestHandler(request -> {
            request.response().end("hello");
        }).listen(8000);
    }
}
{% endhighlight %}

And to run the application:

{% highlight console %}
[root@localhost ~]# vert.x-3.0.0/bin/vertx run WebService.java -instances 2
{% endhighlight %}

The `-instances 2` flag fires up two instances of the Verticle, ensuring we can drive both CPU cores on the Intel NUC.

### Seastar

Seastar is installed and configured using the same instructions in my [previous blog post](http://pseudo.co.de/dpdk-on-an-intel-nuc/).  Note for this test I built Seastar using commit [`696ab29b4ddd0a068d4e8860b7bab5fef658aa87`](https://github.com/cloudius-systems/seastar/commit/696ab29b4ddd0a068d4e8860b7bab5fef658aa87)

After you have built Seastar, to run the sample Seastar web application using the Linux networking stack (note the web application will be reachable on the operating system network IP address):

{% highlight console %}
[root@localhost ~]# /root/seastar/build/release/apps/httpd/httpd --smp 2
EAL: Detected lcore 0 as core 0 on socket 0
EAL: Detected lcore 1 as core 1 on socket 0
EAL: Support maximum 128 logical core(s) by configuration.
EAL: Detected 2 lcore(s)
EAL: VFIO modules not all loaded, skip VFIO support...
EAL: Setting up physically contiguous memory...
EAL: TSC frequency is ~1696075 KHz
EAL: Master lcore 0 is ready (tid=35cd4900;cpuset=[0])
EAL: lcore 1 is ready (tid=2dea6700;cpuset=[1])
EAL: PCI device 0000:00:19.0 on NUMA socket -1
EAL:   probe driver: 8086:1559 rte_em_pmd
EAL:   Not managed by a supported kernel driver, skipped
Seastar HTTP server listening on port 10000 ... 
{% endhighlight %}

The `--smp 2` flag ensures the application will drive both CPU cores.

To run the sample Seastar web application using the DPDK networking stack (note the web application will be reachable on the IP address specified when running the application):

{% highlight console %}
[root@localhost ~]# ifdown eno1
[root@localhost ~]# modprobe uio
[root@localhost ~]# insmod /root/dpdk/x86_64-native-linuxapp-gcc/kmod/igb_uio.ko
[root@localhost ~]# /root/dpdk/tools/dpdk_nic_bind.py --bind=igb_uio eno1
[root@localhost ~]# /root/seastar/build/release/apps/httpd/httpd \
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

## Results

The following table shows the results:

| Concurrent Connections | 50     | 100    | 500    | 1000   |
|:-----------------------|:------:|:------:|:------:|:------:|
| Node.js            | 18532  | 18675  | 17995  | 17746  |
| Go (Gin)           | 59254  | 62147  | 57943  | 57673  |
|----
| Vert.x             | 80855  | 85161  | 90685  | 89824  |
| Seastar (Native)   | 80011  | 85130  | 90353  | 89785  |
| Seastar (DPDK)     | 135358 | 180037 | 190801 | 175099 |
|=====

Each of the columns represents the number of concurrent connections during the test.  For example, for the column titled "50", the corresponding `weighttp` command used is:

{% highlight console %}
weighttp -n 1000000 -c 50 -t 4 -k 192.168.1.200:10000
{% endhighlight %}

And for the column titled "1000":

{% highlight console %}
weighttp -n 1000000 -c 1000 -t 4 -k 192.168.1.200:10000
{% endhighlight %}

The following graph illustrates the results:

<figure>
  <img src="/images/showdown-results.png" alt="Showdown Results">
</figure>

## Wrap Up

Well needless to say, **Seastar with DPDK** blows the competition out of the water!  

The results are impressive, delivering over 2x performance with 100+ concurrent connections when compared with Seastar using the native Linux networking stack.

The performance of Node.js is somewhat pedestrian compared to the rest, but still delivers close to 20,000 requests per second, which is likely more than enough for the majority of web applications you or I will ever write.

Vert.x, which is an application framework I've been working with a lot, performs very well, matching the performance of Seastar (which is a native application written in C++) using the native Linux networking stack and outperforming Go (which again compiles to a native application).  IMHO Vert.x is the most complete of all the application frameworks tested, so all things being equal, Vert.x really is a great choice that performs very well.  Here's hoping in the not too distant future we see Netty (which Vert.x runs on top of) add support for DPDK. 


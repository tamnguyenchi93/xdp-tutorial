<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Lessons</a>
<ul>
<li><a href="#sec-1-1">1.1. Where does AF_XDP performance come from?</a></li>
<li><a href="#sec-1-2">1.2. Details: Actually four SPSC ring queues</a></li>
<li><a href="#sec-1-3">1.3. Gotcha by RX-queue id binding</a></li>
<li><a href="#sec-1-4">1.4. Driver support and zero-copy mode</a></li>
</ul>
</li>
<li><a href="#sec-2">2. Assignments</a>
<ul>
<li><a href="#sec-2-1">2.1. Assignment 1: Run the example program to eat all packets</a></li>
<li><a href="#sec-2-2">2.2. Assignment 2: Write an XDP program to process every other packet</a></li>
<li><a href="#sec-2-3">2.3. Assignment 3: Write an userspace program to reply to IPv6 ping packets</a></li>
</ul>
</li>
</ul>
</div>
</div>


It is important to understand that XDP in-itself is not a kernel bypass
facility. XDP is an **in-kernel** fast-path, that operates on raw-frames
"inline" before they reach the normal Linux Kernel network stack.

To support fast delivery of *raw-frames into userspace*, XDP can **bypass**
the Linux Kernel network stack via XDP\_REDIRECT'ing into a special BPF-map
containing AF\_XDP sockets. The AF\_XDP socket is an new Address Family type.
([The kernel documentation for AF\_XDP](https://www.kernel.org/doc/html/latest/networking/af_xdp.html)).

# Lessons<a id="sec-1" name="sec-1"></a>

## Where does AF\_XDP performance come from?<a id="sec-1-1" name="sec-1-1"></a>

The AF\_XDP socket is really fast, but what the secret behind this
performance boost?

One of the basic ideas behind AF\_XDP dates back to [Van Jacobson](https://en.wikipedia.org/wiki/Van_Jacobson)'s talk about
[network channels](https://lwn.net/Articles/169961/). It is about creating a Lock-free [channel](https://lwn.net/Articles/169961/) directly from
driver RX-queue into an (AF\_XDP) socket.

The basic queues used by AF\_XDP are Single-Producer/Single-Consumer (SPSC)
descriptor ring queues:

-   The **Single-Producer** (SP) bind to specific RX **queue id**, and
    NAPI-softirq assures only 1-CPU process 1-RX-queue id (per scheduler
    interval).

-   The **Single-Consumer** (SC) is one-application, reading descriptors from
    a ring, that point into UMEM area.

There are **no memory allocation** per packet. Instead the UMEM memory area
used for packets is pre-allocated and there by bounded. The UMEM area
consists of a number of equally sized chunks, that userspace have registered
with the kernel (via XDP\_UMEM\_REG setsockopt system call). **Importantly**:
This also means that you are responsible for returning frames to UMEM in
timely manner, and pre-allocated enough for your application usage pattern.

The [transport signature](http://www.lemis.com/grog/Documentation/vj/lca06vj.pdf) that Van Jacobson talked about, are replaced by the
XDP/eBPF program choosing which AF\_XDP socket to XDP\_REDIRECT into.

## Details: Actually four SPSC ring queues<a id="sec-1-2" name="sec-1-2"></a>

As explained in the AF\_XDP kernel doc there are actually 4 SPSC ring queues.

In summary: the AF\_XDP *socket* has two rings for **RX** and **TX**, which
contain descriptors that points into UMEM area. The UMEM area has two rings:
**FILL** ring and **COMPLETION** ring. In the **FILL** ring: the application gives
the kernel a packet area to **RX** fill. In the **COMPLETION** ring, the kernel
tells the application that **TX is done** for a packet area (which then can be
reused). This scheme is for transferring ownership of UMEM packet areas
between the kernel and the userspace application.

## Gotcha by RX-queue id binding<a id="sec-1-3" name="sec-1-3"></a>

The most common mistake: Why am I not seeing any traffic on the AF\_XDP
socket?

As you just learned from above, the AF\_XDP socket bound to a **single
RX-queue id** (for performance reasons). Thus, your userspace program is only
receiving raw-frames from a specific RX-queue id number. NICs will by
default spread flows with RSS-hashing over all available RX-queues. Thus,
traffic likely not hitting queue you expect.

In order to fix that problem, you **MUST** configure the NIC to steer flow to a specific RX-queue.
This can be done via ethtool or TC HW offloading filter setup.

The following example shows how to configure the NIC to steer all UDP ipv4 traffic
to *RX-queue id* 42:

    ethtool -N <interface> flow-type udp4 action 42

The parameter *action* specifies the id of the target *RX-queue*.

In general, a flow rule is composed of a matching criteria and an action.
L2, L3 and L4 header values can be used to specify the matching criteria.
For a comprehensive documentation, please check out the man page of ethtool.
It documents all available header values that can be used as part of the matching criteria.

Alternative work-arounds:
1.  Create as many AF\_XDP sockets as RXQs, and have userspace poll()/select
    on all sockets.
2.  For testing purposes reduce RXQ number to 1,
    e.g. via command `ethtool -L <interface> combined 1`

## Driver support and zero-copy mode<a id="sec-1-4" name="sec-1-4"></a>

As hinted in the intro (driver level) support for AF\_XDP depend on drivers
implementing the XDP\_REDIRECT action. For all driver implementing the basic
XDP\_REDIRECT action, AF\_XDP in "copy-mode" is supported. The "copy-mode" is
surprisingly fast, and does a single-copy of the frame (including any XDP
placed meta-data) into the UMEM area. The userspace API remains the same.

For AF\_XDP "zero-copy" support the driver need to implement and expose the
API for registering and using the UMEM area directly in the NIC RX-ring
structure for DMA delivery.

Depending on your use-case, it can still make sense to use the "copy-mode"
on a "zero-copy" capable driver. If for some-reason, not all traffic on a
RX-queue is for the AF\_XDP socket, and the XDP program multiplex between
XDP\_REDIRECT and XDP\_PASS, then "copy-mode" can be relevant. As in
"zero-copy" mode doing XDP\_PASS have a fairly high cost, which involves
allocating memory and copying over the frame.

# Assignments<a id="sec-2" name="sec-2"></a>

The end goal of this lesson is to build an AF\_XDP program that will send
packets to userspace and if they are IPv6 ping packets reply.

We will do using the automatically installed XDP program, but one of the
assignments it to implement this manually.

## Assignment 1: Run the example program to eat all packets<a id="sec-2-1" name="sec-2-1"></a>

First, you need to set up the test lab environment and start an infinite
ping. You do this by running the following:

    $ eval $(../testenv/testenv.sh alias)
    $ t setup --name veth-adv03
    $ t ping

Now you can start the af\_xdp\_user application and see all the pings being
eaten by it:

    $ sudo ./af_xdp_user -d veth-adv03
    [sudo] password for echaudro:
    AF_XDP RX:             2 pkts (         1 pps)           0 Kbytes (     0 Mbits/s) period:2.000185
           TX:             0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.000185
    
    AF_XDP RX:             4 pkts (         1 pps)           0 Kbytes (     0 Mbits/s) period:2.000152
           TX:             0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.000152

## Assignment 2: Write an XDP program to process every other packet<a id="sec-2-2" name="sec-2-2"></a>

For this exercise, you need to write an eBPF program that will count the
packets received, and use this value to determine if the packet needs to be
sent down the AF\_XDP socket. We want every other packet to be sent to the
AF\_XDP socket.

This should result in every other ping packet being replied too. Here is the
expected output from the ping command, notice the icmp\_seq numbers:

    $ t ping
    Running ping from inside test environment:
    
    PING fc00:dead:cafe:1::1(fc00:dead:cafe:1::1) 56 data bytes
    64 bytes from fc00:dead:cafe:1::1: icmp_seq=2 ttl=64 time=0.038 ms
    64 bytes from fc00:dead:cafe:1::1: icmp_seq=4 ttl=64 time=0.047 ms
    64 bytes from fc00:dead:cafe:1::1: icmp_seq=6 ttl=64 time=0.062 ms
    64 bytes from fc00:dead:cafe:1::1: icmp_seq=8 ttl=64 time=0.083 ms

If you have your custom program ready you can bind it using the &#x2013;filename
option:

    $ sudo ./af_xdp_user -d veth-adv03 --filename af_xdp_kern.o
    AF_XDP RX:             1 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.000171
           TX:             0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.000171
    
    AF_XDP RX:             2 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.000133
           TX:             0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:2.000133

Note that the full solution is included in the af\_xdp\_kern.c file.

## Assignment 3: Write an userspace program to reply to IPv6 ping packets<a id="sec-2-3" name="sec-2-3"></a>

For the final exercise, you need to write some userspace code that will
reply to the ping packets. This needs be done inside the process\_packet()
function.

Once you have done this all pings should receive a reply:

    $ sudo ./af_xdp_user -d veth-adv03
    AF_XDP RX:             2 pkts (         1 pps)           0 Kbytes (     0 Mbits/s) period:2.000175
           TX:             2 pkts (         1 pps)           0 Kbytes (     0 Mbits/s) period:2.000175
    
    AF_XDP RX:             4 pkts (         1 pps)           0 Kbytes (     0 Mbits/s) period:2.000146
           TX:             4 pkts (         1 pps)           0 Kbytes (     0 Mbits/s) period:2.000146
    
    AF_XDP RX:             6 pkts (         1 pps)           0 Kbytes (     0 Mbits/s) period:2.000118
           TX:             6 pkts (         1 pps)           0 Kbytes (     0 Mbits/s) period:2.000118

Note that the full solution is present in the ad\_xdp\_user.c file.
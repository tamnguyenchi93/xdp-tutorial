<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. What you will learn in this lesson</a>
<ul>
<li><a href="#sec-2-1">2.1. Sending packets back to the interface they came from</a></li>
<li><a href="#sec-2-2">2.2. Redirecting packets to other interfaces</a></li>
<li><a href="#sec-2-3">2.3. Building a router using a kernel helper</a></li>
</ul>
</li>
<li><a href="#sec-3">3. Assignments</a>
<ul>
<li><a href="#sec-3-1">3.1. Assignment 1: Send packets back where they came from</a></li>
<li><a href="#sec-3-2">3.2. Assignment 2: Redirect packets between two interfaces</a></li>
<li><a href="#sec-3-3">3.3. Assignment 3: Extend to a bidirectional router</a></li>
<li><a href="#sec-3-4">3.4. Assignment 4: Use the BPF helper for routing</a></li>
</ul>
</li>
</ul>
</div>
</div>


Now that you have come this far, you know how to parse packet data, and how
to modify packets. These are two of the main components of a packet
processing system, but there is one additional component that is missing:
how to redirect packets and transmit them back out onto the network.
This lesson will cover this aspect of packet processing.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   What you will learn in this lesson (See section )
    -   Sending packets back to the interface they came from (See section )
    -   Redirecting packets to other interfaces (See section )
    -   Building a router using a kernel helper (See section )
-   Assignments (See section )
    -   Assignment 1: Send packets back where they came from (See section )
    -   Assignment 2: Redirect packets between two interfaces (See section )
    -   Assignment 3: Extend to a bidirectional router (See section )
    -   Assignment 4: Use the BPF helper for routing (See section )

# What you will learn in this lesson<a id="sec-2" name="sec-2"></a>

## Sending packets back to the interface they came from<a id="sec-2-1" name="sec-2-1"></a>

The `XDP_TX` return value can be used to send the packet back from the same
interface it came from.  This functionality can be used to implement load
balancers, to send simple ICMP replies, etc.  We will use this functionality in
the Assignment 1 to implement a simple ICMP echo server.

Note that in order to the transmit and/or redirect functionality to work, **all**
involved devices should have an attached XDP program, including both veth
peers.  We have to do this because `veth` devices won't deliver
redirected/retransmitted XDP frames unless there is an XDP program attached to
the receiving side of the target `veth` interface. Physical hardware will
likely behave the same. XDP maintainers are currently working on fixing this
behaviour upstream. See the
[Veth XDP: XDP for containers](https://www.netdevconf.org/0x13/session.html?talk-veth-xdp)
talk which describes the reasons behind this problem.  (The `xdpgeneric` mode
may be used without this limitation.)

## Redirecting packets to other interfaces<a id="sec-2-2" name="sec-2-2"></a>

Besides the ability to transmit packets back from the same interface, there is
an option to forward packets to egress ports of other interfaces (if the
corresponding driver supports this feature). This can be done using the
`bpf_redirect` or `bpf_redirect_map` helpers. These helpers will return the
`XDP_REDIRECT` value and this is what should the program return. The
`bpf_redirect` helper actually shouldn't be used in production as it is slow
and can't be configured from user space.  However, we will use it in the
Assignment 2 for the sake of simplicity. To use the `bpf_redirect_map` helper
we need to setup a special map of type `BPF_MAP_TYPE_DEVMAP` which maps virtual
ports to actual network devices. See [this
patch series](https://lwn.net/Articles/728146) which implemented the XDP redirect functionality. An example of
usage can be found in the
[xdp\_redirect\_map\_kern.c](https://github.com/torvalds/linux/blob/master/samples/bpf/xdp_redirect_map_kern.c)
file and the corresponding loader.

## Building a router using a kernel helper<a id="sec-2-3" name="sec-2-3"></a>

As we will see in the Assignments 2 and 3, using the raw redirect functionality
could be quite challenging, as one needs to support a set of data structures
and a custom kernel-to-user space protocol to exchange data.  Fortunately, for
XDP and TC programs we can access the kernel routing table using the
`bpf_fib_lookup()` helper and use this information to redirect packets.  The
Assignment 4 shows how to use this helper to write a program for full
redirection.

One thing to note when using the `bpf_fib_lookup()` function is that it will
return an error code if forwarding is disabled for the ingress interface, so we
need to enable forwarding before running code from the Assignment 4. Note that
enabling forwarding is not required by the `bpf_redirect_map` or `bpf_redirect`
functions, only for `bpf_fib_lookup`.

# Assignments<a id="sec-3" name="sec-3"></a>

## Assignment 1: Send packets back where they came from<a id="sec-3-1" name="sec-3-1"></a>

Use the `XDP_TX` functionality to implement an ICMP/ICMPv6 echo server.
Namely, swap destination and source MAC addresses, then swap IP/IPv6 addresses,
and, finally, replace the ICMP/ICMPv6 `Type` field with an appropriate value.

Note that changing the `Type` field will affect the ICMP checksum. However, as
only a small part of the packet is changing, the Incremental Internet Checksum
(RFC 1624) can be used, so use the `bpf_csum_diff` helper to update the
checksum.

To test the echo server create a new environment with both address families
supported and load the XDP program. Note that we also need to load a dummy
`xdp_pass` program for the peer device as well, as explained in the
Sending packets back to the interface they came from (See section )
section.

    $ t setup -n test --legacy-ip
    $ t exec -n test -- ./xdp_loader -d veth0 -F --progsec xdp_pass
    $ t load -n test -- -F --progsec xdp_icmp_echo

Ping the host and use the `xdp_stat` program to check that the ICMP echo server
actually returned `XDP_TX`. Repeat for both address families (you can pass
the `--legacy-ip` option to the `t ping` command as well):

    $ t ping
    #...
    $ t ping --legacy-ip
    #...
    $ sudo ./xdp_stats -d test
    Collecting stats from BPF map
     - BPF map (bpf_map_type:6) id:115 name:xdp_stats_map key_size:4 value_size:16 max_entries:5
    XDP-action
    XDP_ABORTED            0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250206
    XDP_DROP               0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250262
    XDP_PASS               0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250259
    XDP_TX                 8 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250257
    XDP_REDIRECT           0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250255

## Assignment 2: Redirect packets between two interfaces<a id="sec-3-2" name="sec-3-2"></a>

Two virtual environments are displayed in the picture below named `left` and `right`. Ethernet packets
produced by the `eth1` interface will arrive to the `right` interface and have
the `(dest=Y1,source=Y2)` Ethernet header. Your goal is to redirect these
packets to the `left` interface. Redirected packets will appear on the egress
port of the `left` interface and thus the Ethernet header should be changed to
`(dest=X2,source=X1)` or packets will be dropped by the `eth0` interface.

    Env 1                         Env 2
    ----------------------        ----------------------
    |    eth0 (MAC=X2)   |        |    eth1 (MAC=Y2)   |
    ----------||----------        ----------||----------
        veth0 (MAC=X1)  <-----------  veth1 (MAC=Y1)

Setup the two environments, patch the `xdp_redirect` program accordingly, and
attach it to the `right` interface.  Don't forget to attach a dummy program to
the left inner interface like this:

    $ t exec -n left -- ./xdp_loader -d veth0 -F --progsec xdp_pass

To test load the program, enter the right environment, and ping the inner
interface of the left environment (your IPv6 address may be different):

    $ t enter -n right
    $ ping fc00:dead:cafe:10::2

Run the `tcpdump` program inside the `left` environment. You should see that
the ping requests are delivered and ping replies are sent back. However, unless
forwarding is enabled on the host, they won't be delivered.  (We will fix this
in the next assignment.)

    $ t enter -n left
    # tcpdump -l
    listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
    17:03:11.455320 IP6 fc00:dead:cafe:11::2 > fc00:dead:cafe:10::2: ICMP6, echo request, seq 1, length 64
    17:03:11.455343 IP6 fc00:dead:cafe:10::2 > fc00:dead:cafe:11::2: ICMP6, echo reply, seq 1, length 64

Finally, just in case, check that the `right` environment actually redirects
packets (`XDP_REDIRECT` row should be non-zero):

    $ sudo ./xdp_stats -d right
    
    Collecting stats from BPF map
     - BPF map (bpf_map_type:6) id:183 name:xdp_stats_map key_size:4
       value_size:16 max_entries:5
    XDP-action
    XDP_ABORTED            0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250143
    XDP_DROP               0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250180
    XDP_PASS               0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250180
    XDP_TX                 0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250179
    XDP_REDIRECT         176 pkts (         4 pps)          20 Kbytes (     0 Mbits/s) period:0.250179

## Assignment 3: Extend to a bidirectional router<a id="sec-3-3" name="sec-3-3"></a>

In the previous assignment we were able to pass packets from one interface to
the other. However, we were using the wrong `bpf_redirect` helper and needed to
hardcode interface number and MAC addresses. This is not useful and we will use
better techniques in this Assignment.

This Assignment will show how to use the `bpf_redirect_map` function. Besides
that, to make the program more useful we will use a map which will contain a
mapping between source and destination MAC addresses. The actual goal of this
Assignment is to write a user-space helper which will configure these maps
after loading the program, as the XDP part is pretty simple.  To do this patch
the `xdp_prog_user.c` program.

To test the code, configure environment as in the Assignment 2 and install the
`xdp_redirect_map` program on both interfaces:

    $ t load -n left -- -F --progsec xdp_redirect_map
    $ t load -n right -- -F --progsec xdp_redirect_map

Don't forget about dummy programs for inner interfaces:

    $ t exec -n left -- ./xdp_loader -d veth0 -F --progsec xdp_pass
    $ t exec -n right -- ./xdp_loader -d veth0 -F --progsec xdp_pass

Configure parameters for both interfaces using the new `xdp_prog_user` helper.
For simplicity there is a new special helper, `t redirect`, which will
do the work for you.  See its implementation to see how it obtains inner MAC
addresses by interface names.

    $ t redirect right left

Pings between the two inner interfaces should pass now. Check that they are
actually forwarded by our programs by running `xdp_stats` on both interfaces:

    $ sudo ./xdp_stats -d right
    
    Collecting stats from BPF map
     - BPF map (bpf_map_type:6) id:183 name:xdp_stats_map key_size:4 value_size:16 max_entries:5
    XDP-action
    XDP_ABORTED            0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250185
    XDP_DROP               0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250239
    XDP_PASS               0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250234
    XDP_TX                 0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250231
    XDP_REDIRECT        1303 pkts (         0 pps)         153 Kbytes (     0 Mbits/s) period:0.250228
    
    ^C
    $ sudo ./xdp_stats -d left
    
    Collecting stats from BPF map
     - BPF map (bpf_map_type:6) id:186 name:xdp_stats_map key_size:4 value_size:16 max_entries:5
    XDP-action
    XDP_ABORTED            0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250154
    XDP_DROP               0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250206
    XDP_PASS               0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250206
    XDP_TX                 0 pkts (         0 pps)           0 Kbytes (     0 Mbits/s) period:0.250206
    XDP_REDIRECT          22 pkts (         0 pps)           2 Kbytes (     0 Mbits/s) period:0.250206
    
    ^C

If, however, we try to ping outer interfaces from inner or vice versa, we
won't see any replies, as packets destined to outer interfaces will also be
redirected. Besides that, our implementation doesn't easily scale to more
than two interfaces. The next assignment will show how to forward packets in
a better manner using a kernel helper.

## Assignment 4: Use the BPF helper for routing<a id="sec-3-4" name="sec-3-4"></a>

After completing Assignment 3, you'll have a hard-coded redirect between the
two inner interfaces. As was mentioned above, we are able to deliver packets
only between inner interfaces—a packet destined to an outer interface will be
delivered to the opposite inner interface and dropped there because of the
wrong destination L3 address. We can manually check the IP/IPv6 addresses
and return `XDP_PASS` when packets destined to outer interfaces, but this
doesn't cover all cases and wouldn't it be better to dynamically lookup
where each packet should go?

This assignment teaches to use the `bpf_fib_lookup` helper. This function lets
XDP and TC programs to access the kernel routing table and will return an
ifindex of interface to forward packet to.  As was noted the `bpf_redirect_map`
function accepts virtual ports, so to use ifindexes you will need to update the
`xdp_prog_user.c` program to setup 1-1 mapping between virtual ports and
network devices where each virtual port corresponds to a device with the same
ifindex.

The Assignment, for the most part, reproduces the
[xdp\_fwd\_kern.c](https://github.com/torvalds/linux/blob/master/samples/bpf/xdp_fwd_kern.c)
example from the Linux kernel, but was patched to update statistics as in other
examples and to check all the return values from the `bpf_fib_lookup()`
function.

To test the router check that you can get a ping and/or establish
a TCP connection between any two interfaces: inner-inner, inner-outer,
outer-outer. Programs should return `XDP_PASS` for inner-outer communications
and `XDP_FORWARD` for inner-inner.

Try more than two test environments. Run the `xdp_stats` program to verify that
this is the XDP program does forwarding, and not the network stack (as
forwarding should be enabled for this Assignment as noted above). Don't forget
to enable forwarding for the interfaces.

    $ t setup -n uno --legacy-ip
    $ t setup -n dos --legacy-ip
    $ t setup -n tres --legacy-ip
    
    $ sudo sysctl net.ipv4.conf.all.forwarding=1
    $ sudo sysctl net.ipv6.conf.all.forwarding=1
    
    $ t load -n uno -- -F --progsec xdp_router
    $ t load -n dos -- -F --progsec xdp_router
    $ t load -n tres -- -F --progsec xdp_router
    
    $ t exec -n uno -- ./xdp_loader -d veth0 -F --progsec xdp_pass
    $ t exec -n dos -- ./xdp_loader -d veth0 -F --progsec xdp_pass
    $ t exec -n tres -- ./xdp_loader -d veth0 -F --progsec xdp_pass
    
    $ sudo ./xdp_prog_user -d uno
    $ sudo ./xdp_prog_user -d dos
    $ sudo ./xdp_prog_user -d tres
    
    $ sudo ./xdp_stats -d uno
    $ sudo ./xdp_stats -d dos
    $ sudo ./xdp_stats -d tres
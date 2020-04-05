<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. What you will learn in this lesson</a>
<ul>
<li><a href="#sec-2-1">2.1. Rewriting packet data with direct memory access</a></li>
<li><a href="#sec-2-2">2.2. Enlarging and shrinking packet size</a></li>
</ul>
</li>
<li><a href="#sec-3">3. Assignments</a>
<ul>
<li><a href="#sec-3-1">3.1. Assignment 1: Rewrite port numbers</a></li>
<li><a href="#sec-3-2">3.2. Assignment 2: Remove the outermost VLAN tag</a></li>
<li><a href="#sec-3-3">3.3. Assignment 3: Add back a missing VLAN tag</a></li>
</ul>
</li>
</ul>
</div>
</div>


Having completed the packet parsing lesson in packet01, you are now familiar
with how to structure packet parsing, how to make sure you do proper bounds
checking before referencing packet data, and how to decide the final packet
verdict with return codes. In this lesson we build on this to show how to
modify the packet contents.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   What you will learn in this lesson (See section )
    -   Rewriting packet data with direct memory access (See section )
    -   Enlarging and shrinking packet size (See section )
-   Assignments (See section )
    -   Assignment 1: Rewrite port numbers (See section )
    -   Assignment 2: Remove the outermost VLAN tag (See section )
    -   Assignment 3: Add back a missing VLAN tag (See section )

# What you will learn in this lesson<a id="sec-2" name="sec-2"></a>

## Rewriting packet data with direct memory access<a id="sec-2-1" name="sec-2-1"></a>

As we saw in the previous lesson, the verifier will check that all memory
accesses to packet data first perform correct bounds checking to make sure
it doesn't reference memory outside the packet. This applies not only to
packet data reads, but also to writes; which means that we can rewrite
packet data simply by changing the memory it occupies. We will use this in
the assignments below to modify packet header fields.

## Enlarging and shrinking packet size<a id="sec-2-2" name="sec-2-2"></a>

While many things can be accomplished simply by rewriting existing packet
data, sometimes it is necessary to add or remove chunks of memory entirely,
for instance to perform encapsulation, or to remove protocol headers from a
packet. The kernel exposes an eBPF helper function to achieve this, which is
called `bpf_xdp_adjust_head()`. This function takes the XDP context object
and an adjustment size as parameter, and will move the header pointer by
this many bytes (i.e., a positive number will shrink the size of the packet
data, while a negative number will add that many bytes to the front of the
packet).

There are a few things to be aware of when using this helper:

1.  First, it may fail either because the adjustment would make the packet
    data too small to contain at least an Ethernet header, or because there
    is not enough space in memory before the start of the packet (packet data
    is put at a fixed offset from the start of the memory page).

2.  Second, the helper only adjusts the data pointer. It is up to the XDP
    program to ensure that the packet data is valid afterwards. This typically
    involves rewriting the Ethernet header at the new start of the packet
    location, and adjusting any subsequent header fields to match.

3.  Finally, the verifier will discard all information about previous bounds
    checks after the packet size has been adjusted. This means that the XDP
    program needs to perform new bounds checks **including re-evaluating the
    data and data\_end pointers** after adjusting the packet size.

There is also a `bpf_xdp_adjust_tail()` which can be used to move the end of
the packet data, but this only supports shrinking the packet, not extending it;
but otherwise it functions identically to `bpf_xdp_adjust_head()`.

# Assignments<a id="sec-3" name="sec-3"></a>

In this lesson we will be creating two programs: One that rewrites the
destination port number of TCP and UDP packets to be one less than its
original. And another that removes the outermost VLAN encapsulation header
if one exists, or add a new one if it doesn't.

## Assignment 1: Rewrite port numbers<a id="sec-3-1" name="sec-3-1"></a>

For this assignment you will need to parse the TCP and UDP headers and
rewrite the port number before passing on the packet. These headers are
defined in `<linux/tcp.h>` and `<linux/udp.h>`, respectively.

Rewriting is simply a matter of writing to the right field in the header
(after parsing it). E.g.:

    udphdr->dest = bpf_htons(bpf_ntohs(udphdr->dest) - 1);

You can use `tcpdump` to verify that this works. As a packet generator you
can use the `socat` utility. The following will generate a UDP packet to
port 2000 for each line you type on stdin:

    $ t exec -- socat - 'udp6:[fc00:dead:cafe:1::1]:2000'

You can view these with `tcpdump`:

    $ t tcpdump
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on xdptut-3c93, link-type EN10MB (Ethernet), capture size 262144 bytes
    12:54:31.085948 d2:9e:c0:4f:3b:7b > 32:71:5a:a4:74:c1, ethertype IPv6 (0x86dd), length 67: fc00:dead:cafe:1::2.35126 > fc00:dead:cafe:1::1.2000: UDP, length 5

When your program is working correctly, the destination port (2000 near the
end of the line) should be 1999 instead.

## Assignment 2: Remove the outermost VLAN tag<a id="sec-3-2" name="sec-3-2"></a>

In this assignment we will start implementing the program that removes the
outermost VLAN tag if one exists. To do this, fill in the `vlan_tag_pop()`
function that is prototyped in <xdp_prog_kern.c>. The function prototype
contains the variable definitions and inline comments from our solution to
the issue, to guide you in the implementation.

Once you have implemented the function, test that it works by setting up a
test environment with VLANs enabled (like in the previous lesson), and run
`t ping --vlan` in one window, while looking at the output of `t tcpdump` in
another. You should see no VLAN tags on the echo request packets; the echo
replies will still have VLAN tags, because the kernel will reply to the ping
even though it is targeting a different interface, and the replies will be
routed out the interface that actually has the IP address being pinged
(i.e., the virtual VLAN interface).

## Assignment 3: Add back a missing VLAN tag<a id="sec-3-3" name="sec-3-3"></a>

In this assignment we will implement the opposite of the previous one: I.e.,
the code that adds a VLAN tag if none exists. Just hardcode the VLAN ID to a
value of your choosing; and test the program the same way as with the
previous assignment (but run `t ping` without the `--vlan` parameter, and
verify that the ICMP echo request packets do have a VLAN tag added to them).
<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. Solutions</a>
<ul>
<li><a href="#sec-2-1">2.1. Packet01: packet parsing</a>
<ul>
<li><a href="#sec-2-1-1">2.1.1. Assignment 1: Fix the bounds checking error</a></li>
<li><a href="#sec-2-1-2">2.1.2. Assignment 2: Parsing the IP header</a></li>
<li><a href="#sec-2-1-3">2.1.3. Assignment 3: Parsing the ICMPv6 header and reacting to it</a></li>
<li><a href="#sec-2-1-4">2.1.4. Assignment 4: Adding VLAN support</a></li>
<li><a href="#sec-2-1-5">2.1.5. Assignment 5: Adding IPv4 support</a></li>
</ul>
</li>
<li><a href="#sec-2-2">2.2. Packet02: packet rewriting</a>
<ul>
<li><a href="#sec-2-2-1">2.2.1. Assignment 1: Rewrite port numbers</a></li>
<li><a href="#sec-2-2-2">2.2.2. Assignment 2: Remove the outermost VLAN tag</a></li>
<li><a href="#sec-2-2-3">2.2.3. Assignment 3: Add back a missing VLAN tag</a></li>
</ul>
</li>
<li><a href="#sec-2-3">2.3. Packet03: redirecting packets</a>
<ul>
<li><a href="#sec-2-3-1">2.3.1. Assignment 1: Send packets back where they came from</a></li>
<li><a href="#sec-2-3-2">2.3.2. Assignment 2: Redirect packets between two interfaces</a></li>
<li><a href="#sec-2-3-3">2.3.3. Assignment 3: Extend to a bidirectional router</a></li>
<li><a href="#sec-2-3-4">2.3.4. Assignment 4: Use the BPF helper for routing</a></li>
</ul>
</li>
</ul>
</li>
</ul>
</div>
</div>


This directory contains solutions to all the assignments in the
[packet01](../packet01-parsing/),
[packet02](../packet02-rewriting/), and
[packet03](../packet03-redirecting/) lessons.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   Solutions (See section )
    -   Packet01: packet parsing (See section )
    -   Packet02: packet rewriting (See section )
    -   Packet03: redirecting packets (See section )

# Solutions<a id="sec-2" name="sec-2"></a>

## Packet01: packet parsing<a id="sec-2-1" name="sec-2-1"></a>

### Assignment 1: Fix the bounds checking error<a id="sec-2-1-1" name="sec-2-1-1"></a>

See the `parse_ethhdr` function from the [parsing\_helpers.h](../common/parsing_helpers.h) file.

### Assignment 2: Parsing the IP header<a id="sec-2-1-2" name="sec-2-1-2"></a>

See the `parse_ip6hdr` function from the [parsing\_helpers.h](../common/parsing_helpers.h) file.

### Assignment 3: Parsing the ICMPv6 header and reacting to it<a id="sec-2-1-3" name="sec-2-1-3"></a>

See the `parse_icmp6hdr` function from the [parsing\_helpers.h](../common/parsing_helpers.h)
file.  The sequence number should be accessed as `bpf_ntohs(icmp6h->icmp6_sequence)`
as it is a 2-byte value in the network order.

### Assignment 4: Adding VLAN support<a id="sec-2-1-4" name="sec-2-1-4"></a>

See the `parse_ethhdr` function from the [parsing\_helpers.h](../common/parsing_helpers.h) file.

### Assignment 5: Adding IPv4 support<a id="sec-2-1-5" name="sec-2-1-5"></a>

See the `parse_iphdr` and `parse_icmphdr` functions from the [parsing\_helpers.h](../common/parsing_helpers.h) file.

## Packet02: packet rewriting<a id="sec-2-2" name="sec-2-2"></a>

### Assignment 1: Rewrite port numbers<a id="sec-2-2-1" name="sec-2-2-1"></a>

An example XDP program can be found in the `xdp_patch_ports` section in the [xdp\_prog\_kern\_02.c](xdp_prog_kern_02.c) file. The program will decrease by one destination port number in any TCP or UDP packet.

### Assignment 2: Remove the outermost VLAN tag<a id="sec-2-2-2" name="sec-2-2-2"></a>

See the `vlan_tag_pop` function from the [rewrite\_helpers.h](../common/rewrite_helpers.h) file.
An example XDP program can be found in the `xdp_vlan_swap` section in the [xdp\_prog\_kern\_02.c](xdp_prog_kern_02.c) file.

### Assignment 3: Add back a missing VLAN tag<a id="sec-2-2-3" name="sec-2-2-3"></a>

See the `vlan_tag_push` function from the [rewrite\_helpers.h](../common/rewrite_helpers.h) file.
An example XDP program can be found in the `xdp_vlan_swap` section in the [xdp\_prog\_kern\_02.c](xdp_prog_kern_02.c) file.

## Packet03: redirecting packets<a id="sec-2-3" name="sec-2-3"></a>

### Assignment 1: Send packets back where they came from<a id="sec-2-3-1" name="sec-2-3-1"></a>

See the `xdp_icmp_echo` program in the [xdp\_prog\_kern\_03.c](xdp_prog_kern_03.c) file.

### Assignment 2: Redirect packets between two interfaces<a id="sec-2-3-2" name="sec-2-3-2"></a>

See the `xdp_redirect` program in the [xdp\_prog\_kern\_03.c](xdp_prog_kern_03.c) file.

### Assignment 3: Extend to a bidirectional router<a id="sec-2-3-3" name="sec-2-3-3"></a>

See the `xdp_redirect_map` program in the [xdp\_prog\_kern\_03.c](xdp_prog_kern_03.c) file.
Userspace part of the assignment is implemented in the [xdp\_prog\_user.c](xdp_prog_user.c) file.

### Assignment 4: Use the BPF helper for routing<a id="sec-2-3-4" name="sec-2-3-4"></a>

See the `xdp_router` program in the [xdp\_prog\_kern\_03.c](xdp_prog_kern_03.c) file.
Userspace part of the assignment is implemented in the [xdp\_prog\_user.c](xdp_prog_user.c) file.
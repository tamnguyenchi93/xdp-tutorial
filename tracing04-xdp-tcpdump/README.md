<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. Dump the packet sample</a></li>
<li><a href="#sec-3">3. Assignments</a>
<ul>
<li><a href="#sec-3-1">3.1. Assignment 1: Setting up your test lab</a></li>
<li><a href="#sec-3-2">3.2. Assignment 2: The PCAP dump file</a></li>
</ul>
</li>
</ul>
</div>
</div>


In this lesson we will show how to dump the packet samples
from XDP program all the way to the pcap dump file.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   Dump the packet sample (See section )
-   Assignments (See section )
    -   Assignment 1: Setting up your test lab (See section )
    -   Assignment 2: The PCAP dump file (See section )

# Dump the packet sample<a id="sec-2" name="sec-2"></a>

In this example we will show how to send data and packet sample
into user space via perf event.

First you need to define the event map, which will allow you
to send events to the user space via perf event ring buffer:

    struct bpf_map_def SEC("maps") my_map = {
            .type           = BPF_MAP_TYPE_PERF_EVENT_ARRAY,
            .key_size       = sizeof(int),
            .value_size     = sizeof(__u32),
            .max_entries    = MAX_CPUS,
    };

The value\_size determines the size of the event data we will
be posting to the user space through perf event ring buffer.
In our case it's sizeof(u32) which is size of following structure:

    struct S {
            __u16 cookie;
            __u16 pkt_len;
    } __packed;

We set the values of the event (metadata variable) and pass them
into the bpf\_perf\_event\_output call:

    int xdp_sample_prog(struct xdp_md *ctx)
    {
            void *data_end = (void *)(long)ctx->data_end;
            void *data = (void *)(long)ctx->data;
    
            ...
    
            __u64 flags = BPF_F_CURRENT_CPU;
            struct S metadata;
    
            metadata.cookie = 0xdead;
            metadata.pkt_len = (__u16)(data_end - data);
    
            ret = bpf_perf_event_output(ctx, &my_map, flags,
                                        &metadata, sizeof(metadata));
            ...

To add the actual packet dump to the event, we can
set flags upper 32 bits with the size of the requested sample
and the bpf\_perf\_event\_output will attach the specified
amount of bytes from packet to the perf event:

    __u64 flags = BPF_F_CURRENT_CPU;
    
    flags |= (__u64)sample_size << 32;
    
    ret = bpf_perf_event_output(ctx, &my_map, flags,
                                &metadata, sizeof(metadata));

Please check the whole eBPF code in xdp\_sample\_pkts\_kern.c file.

# Assignments<a id="sec-3" name="sec-3"></a>

## Assignment 1: Setting up your test lab<a id="sec-3-1" name="sec-3-1"></a>

In this lesson we will use the setup of the previous lesson:
Basic02 - loading a program by name <https://github.com/xdp-project/xdp-tutorial/tree/master/basic02-prog-by-name#assignment-2-add-xdp_abort-program>

    $ sudo ../testenv/testenv.sh setup --name veth-basic02

and make some packets:

    $ sudo ../testenv/testenv.sh enter --name veth-basic02
    # ping  fc00:dead:cafe:1::1
    PING fc00:dead:cafe:1::1(fc00:dead:cafe:1::1) 56 data bytes

## Assignment 2: The PCAP dump file<a id="sec-3-2" name="sec-3-2"></a>

Build the `xdp_sample_pkts_user` dump program; to do so you might have to
install the `libpcap-dev` and the 32 bit libc dev packages.  Load the eBPF
kernel packets dump program and store the packets to the dump file:

    $ sudo ./xdp_sample_pkts_user -d veth-basic02 -F
    pkt len: 118   bytes. hdr: 76 58 28 55 df 4e fa e2 b6 27 8e 79 86 dd 60 0d 48 1b 00 40 3a 40 fc 00 de ad ca fe 00 ...
    pkt len: 118   bytes. hdr: 76 58 28 55 df 4e fa e2 b6 27 8e 79 86 dd 60 0d 48 1b 00 40 3a 40 fc 00 de ad ca fe 00 ...
    ^C
    2 packet samples stored in samples.pcap

Check the pcap dump with the tcpdump application:

    $ tcpdump -r ./samples.pcap
    reading from file ./samples.pcap, link-type EN10MB (Ethernet)
    12:12:04.553039 IP6 fc00:dead:cafe:1::2 > krava: ICMP6, echo request, seq 2177, length 64
    12:12:05.576864 IP6 fc00:dead:cafe:1::2 > krava: ICMP6, echo request, seq 2178, length 64
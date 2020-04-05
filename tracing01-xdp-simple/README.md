<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. XDP tracepoints</a>
<ul>
<li><a href="#sec-2-1">2.1. Tracepoint program section</a></li>
<li><a href="#sec-2-2">2.2. Tracepoint arguments</a></li>
<li><a href="#sec-2-3">2.3. Tracepoint attaching</a></li>
</ul>
</li>
<li><a href="#sec-3">3. HASH map</a></li>
<li><a href="#sec-4">4. Assignments</a>
<ul>
<li><a href="#sec-4-1">4.1. Assignment 1: Setting up your test lab</a></li>
<li><a href="#sec-4-2">4.2. Assignment 2: Load tracepoint monitor program</a></li>
</ul>
</li>
</ul>
</div>
</div>


In this lesson we will show how to create and load eBPF program that
hooks on xdp:exception tracepoint and get its values to user space
stats application.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   XDP tracepoints (See section )
    -   Tracepoint program section (See section )
    -   Tracepoint arguments (See section )
    -   Tracepoint attaching (See section )
-   HASH map (See section )
-   Assignments (See section )
    -   Assignment 1: Setting up your test lab (See section )
    -   Assignment 2: Load tracepoint monitor program (See section )

# XDP tracepoints<a id="sec-2" name="sec-2"></a>

The eBPF programs can be attached also to tracepoints. There are
several tracepoints related to the xdp tracepoint subsystem:

    ls /sys/kernel/debug/tracing/events/xdp/
    xdp_cpumap_enqueue
    xdp_cpumap_kthread
    xdp_devmap_xmit
    xdp_exception
    xdp_redirect
    xdp_redirect_err
    xdp_redirect_map
    xdp_redirect_map_err

## Tracepoint program section<a id="sec-2-1" name="sec-2-1"></a>

The bpf library expects the tracepoint eBPF program to be stored
in a section with following name:

    tracepoint/<sys>/<tracepoint>

where `<sys>` is the tracepoint subsystem and `<tracepoint>` is
the tracepoint name, which can be done with following construct:

    SEC("tracepoint/xdp/xdp_exception")
    int trace_xdp_exception(struct xdp_exception_ctx *ctx)

## Tracepoint arguments<a id="sec-2-2" name="sec-2-2"></a>

There's single program pointer argument which points
to the structure, that defines the tracepoint fields.

Like for xdp:xdp\_exception tracepoint:

    struct xdp_exception_ctx {
            __u64 __pad;      // First 8 bytes are not accessible by bpf code
            __s32 prog_id;    //      offset:8;  size:4; signed:1;
            __u32 act;        //      offset:12; size:4; signed:0;
            __s32 ifindex;    //      offset:16; size:4; signed:1;
    };
    
    int trace_xdp_exception(struct xdp_exception_ctx *ctx)

This struct is exported in tracepoint format file:

    # cat /sys/kernel/debug/tracing/events/xdp/xdp_exception/format
    ...
            field:unsigned short common_type;       offset:0;       size:2; signed:0;
            field:unsigned char common_flags;       offset:2;       size:1; signed:0;
            field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
            field:int common_pid;   offset:4;       size:4; signed:1;
    
            field:int prog_id;      offset:8;       size:4; signed:1;
            field:u32 act;  offset:12;      size:4; signed:0;
            field:int ifindex;      offset:16;      size:4; signed:1;
    ...

## Tracepoint attaching<a id="sec-2-3" name="sec-2-3"></a>

To load a tracepoint program for this example we use following bpf
library helper function:

    bpf_prog_load(cfg->filename, BPF_PROG_TYPE_TRACEPOINT, &obj, &bpf_fd))

It loads all the programs from the object together with maps and
returns file descriptor of the first one.

To attach the program to the tracepoint we need to create a tracepoint
perf event and attach the eBPF program to it, using its file descriptor
via PERF\_EVENT\_IOC\_SET\_BPF ioctl call:

    err = ioctl(fd, PERF_EVENT_IOC_SET_BPF, bpf_fd);

Please check trace\_load\_and\_stats.c load\_bpf\_and\_trace\_attach function
for all the details.

# HASH map<a id="sec-3" name="sec-3"></a>

This example is using PERCPU HASH map, that stores number of aborted
packets for interface

    struct bpf_map_def SEC("maps") xdp_stats_map = {
            .type        = BPF_MAP_TYPE_PERCPU_HASH,
            .key_size    = sizeof(__s32),
            .value_size  = sizeof(__u64),
            .max_entries = 10,
    };

The interface is similar to the ARRAY map except that we need to specifically
create new element in the hash if it does not exist:

    /* Lookup in kernel BPF-side returns pointer to actual data. */
    valp = bpf_map_lookup_elem(&xdp_stats_map, &key);
    
    /* If there's no record for interface, we need to create one,
     * with number of packets == 1
     */
    if (!valp) {
            __u64 one = 1;
            return bpf_map_update_elem(&xdp_stats_map, &key, &one, 0) ? 1 : 0;
    }
    
    (*valp)++;

Please check trace\_prog\_kern.c for the full code.

# Assignments<a id="sec-4" name="sec-4"></a>

## Assignment 1: Setting up your test lab<a id="sec-4-1" name="sec-4-1"></a>

In this lesson we will use the setup of the previous lesson:
Basic02 - loading a program by name <https://github.com/xdp-project/xdp-tutorial/tree/master/basic02-prog-by-name#assignment-2-add-xdp_abort-program>

and load XDP program from xdp\_prog\_kern.o that will abort every
incoming packet:

    SEC("xdp_abort")
    int xdp_drop_func(struct xdp_md *ctx)
    {
            return XDP_ABORTED;
    }

with xdp\_loader from previous lessson:
Assignment 2: Add xdp\_abort program <https://github.com/xdp-project/xdp-tutorial/tree/master/basic02-prog-by-name#assignment-2-add-xdp_abort-program>

Setup the environment:

    $ sudo ../testenv/testenv.sh setup --name veth-basic02

Load the XDP program, tak produces aborted packets:

    $ sudo ./xdp_loader --dev veth-basic02 --force --progsec xdp_abort

and make some packets:

    $ sudo ../testenv/testenv.sh enter --name veth-basic02
    # ping  fc00:dead:cafe:1::1
    PING fc00:dead:cafe:1::1(fc00:dead:cafe:1::1) 56 data bytes

## Assignment 2: Load tracepoint monitor program<a id="sec-4-2" name="sec-4-2"></a>

Now when you run the trace\_load\_and\_stats application it will
load and attach the tracepoint eBPF program and display number
of aborted packets per interface:

    # ./trace_load_and_stats
    Success: Loaded BPF-object(trace_prog_kern.o)
    
    Collecting stats from BPF map
     - BPF map (bpf_map_type:1) id:46 name:xdp_stats_map key_size:4 value_size:4 max_entries:10
    
    veth-basic02 (2)
    veth-basic02 (4)
    veth-basic02 (6)
    ...
<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. RAW tracepoints</a></li>
<li><a href="#sec-3">3. Assignments</a>
<ul>
<li><a href="#sec-3-1">3.1. Assignment 1: Monitor all xdp tracepoints</a></li>
</ul>
</li>
</ul>
</div>
</div>


In this lesson we will show how to attach to and monitor all
xdp related raw tracepoints and some related info to user space
stat application.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   RAW tracepoints (See section )
-   Assignments (See section )
    -   Assignment 1: Monitor all xdp tracepoints (See section )

# RAW tracepoints<a id="sec-2" name="sec-2"></a>

The RAW tracepoint is eBPF alternative to standard tracepoint,
which allows attaching eBPF program to tracepoint without the
perf layer being involved/executed.

The bpf library expects the raw tracepoint eBPF program to be stored
in a section with following name:

    raw_tracepoint/<sys>/<tracepoint>

where `<sys>` is the tracepoint subsystem and `<tracepoint>` is
the tracepoint name, which can be done with following construct:

    SEC("raw_tracepoint/xdp/xdp_exception")
    int trace_xdp_exception(struct xdp_exception_ctx *ctx)

The libbpf library exports interface to load and attach raw tracepoint
programs, following call will load every program in the `cfg->filename`
object as raw tracepoint programs:

    err = bpf_prog_load(cfg->filename, BPF_PROG_TYPE_RAW_TRACEPOINT, &obj, &bpf_fd));

You can then iterate through all the programs and attach
every program to the raw tracepoint:

    bpf_object__for_each_program(prog, obj) {
            ...
            bpf_fd = bpf_program__fd(prog);
            ...
            err = bpf_raw_tracepoint_open(tp, bpf_fd);
            ...
    }

for more details please check load\_bpf\_and\_trace\_attach function
in trace\_load\_and\_stats.c object.

# Assignments<a id="sec-3" name="sec-3"></a>

## Assignment 1: Monitor all xdp tracepoints<a id="sec-3-1" name="sec-3-1"></a>

    $ sudo ./trace_load_and_stats
    XDP-event       CPU:to  pps          drop-pps     extra-info
    XDP_REDIRECT    total   0            0            Success
    XDP_REDIRECT    total   0            0            Error
    Exception       0       0            11           XDP_UNKNOWN
    Exception       1       0            2            XDP_UNKNOWN
    Exception       2       0            36           XDP_UNKNOWN
    Exception       3       0            29           XDP_UNKNOWN
    Exception       4       0            3            XDP_UNKNOWN
    Exception       5       0            8            XDP_UNKNOWN
    Exception       total   0            91           XDP_UNKNOWN
    cpumap-kthread  total   0            0            0          
    devmap-xmit     total   0            0            0.00
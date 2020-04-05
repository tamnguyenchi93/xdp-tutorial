<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. Solutions</a>
<ul>
<li><a href="#sec-2-1">2.1. Basic01: loading your first BPF program</a></li>
<li><a href="#sec-2-2">2.2. Basic02: loading a program by name</a>
<ul>
<li><a href="#sec-2-2-1">2.2.1. Assignment 1: Setting up your test lab</a></li>
<li><a href="#sec-2-2-2">2.2.2. Assignment 2: Add xdp_abort program</a></li>
</ul>
</li>
<li><a href="#sec-2-3">2.3. Basic03: counting with BPF maps</a></li>
<li><a href="#sec-2-4">2.4. Basic04: pinning of maps</a>
<ul>
<li><a href="#sec-2-4-1">2.4.1. Assignment 1: (xdp_stats.c) reload map file-descriptor</a></li>
<li><a href="#sec-2-4-2">2.4.2. Assignment 2: (xdp_loader.c) reuse pinned map</a></li>
</ul>
</li>
</ul>
</li>
</ul>
</div>
</div>


This directory contains solutions to all the assignments in the
[basic01](../basic01-xdp-pass/),
[basic02](../basic02-prog-by-name/),
[basic03](../basic03-map-counter/), and
[basic04](../basic04-pinning-maps/) lessons.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   Solutions (See section )
    -   Basic01: loading your first BPF program (See section )
    -   Basic02: loading a program by name (See section )
    -   Basic03: counting with BPF maps (See section )
    -   Basic04: pinning of maps (See section )

# Solutions<a id="sec-2" name="sec-2"></a>

## Basic01: loading your first BPF program<a id="sec-2-1" name="sec-2-1"></a>

This lesson doesn't contain any assignments except to repeat the steps listed
in the lesson readme file.

## Basic02: loading a program by name<a id="sec-2-2" name="sec-2-2"></a>

### Assignment 1: Setting up your test lab<a id="sec-2-2-1" name="sec-2-2-1"></a>

No code is needed, just repeat the steps listed in the assignment description.

### Assignment 2: Add xdp\_abort program<a id="sec-2-2-2" name="sec-2-2-2"></a>

Just add the following section to the
[xdp\_prog\_kern.c](../basic02-prog-by-name/xdp_prog_kern.c) program and
follow the steps listed in the assignment description:

    SEC("xdp_abort")
    int  xdp_abort_func(struct xdp_md *ctx)
    {
        return XDP_ABORTED;
    }

## Basic03: counting with BPF maps<a id="sec-2-3" name="sec-2-3"></a>

The solutions to all three assignments can be found in the following files:

-   The [common\_kern\_user.h](../basic04-pinning-maps/common_kern_user.h) file contains the new structure `datarec` definition.
-   The [xdp\_prog\_kern.c](../basic04-pinning-maps/xdp_prog_kern.c) file contains the new `xdp_stats_map` map definition and the updated `xdp_stats_record_action` function.

Note that for use in later lessons/assignments the code was moved to the following files:
[xdp\_stats\_kern\_user.h](../common/xdp_stats_kern_user.h) and
[xdp\_stats\_kern.h](../common/xdp_stats_kern.h). So in order to use the
`xdp_stats_record_action` function in later XDP programs, just include the
following header files:

    #include "../common/xdp_stats_kern_user.h"
    #include "../common/xdp_stats_kern.h"

For a user-space application, only the former header is needed.

## Basic04: pinning of maps<a id="sec-2-4" name="sec-2-4"></a>

### Assignment 1: (xdp\_stats.c) reload map file-descriptor<a id="sec-2-4-1" name="sec-2-4-1"></a>

See the [xdp\_stats.c](xdp_stats.c) program in this directory.

### Assignment 2: (xdp\_loader.c) reuse pinned map<a id="sec-2-4-2" name="sec-2-4-2"></a>

See the [xdp\_loader.c](xdp_loader.c) program in this directory.
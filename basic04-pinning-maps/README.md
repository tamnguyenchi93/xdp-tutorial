<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. Solutions to basic03 assignments</a></li>
<li><a href="#sec-3">3. What you will learn in this lesson</a>
<ul>
<li><a href="#sec-3-1">3.1. bpf syscall wrappers</a></li>
<li><a href="#sec-3-2">3.2. Mounting the BPF file system</a></li>
<li><a href="#sec-3-3">3.3. Gotchas with pinned maps</a></li>
<li><a href="#sec-3-4">3.4. Deleting maps on XDP reload</a>
<ul>
<li><a href="#sec-3-4-1">3.4.1. Reusing maps with libbpf</a></li>
</ul>
</li>
</ul>
</li>
<li><a href="#sec-4">4. Assignments</a>
<ul>
<li><a href="#sec-4-1">4.1. Assignment 1: (xdp_stats.c) reload map file-descriptor</a></li>
<li><a href="#sec-4-2">4.2. Assignment 2: (xdp_loader.c) reuse pinned map</a></li>
</ul>
</li>
</ul>
</div>
</div>


In this lesson you will learn about reading BPF maps from another "external"
program.

In basic03 the [xdp\_load\_and\_stats.c](../basic03-map-counter/xdp_load_and_stats.c) program was both loading the BPF program
and reading the stats from the map. This was practical as the map
file descriptor was readily available; however it is often limiting that a
map can only be accessed from the same program that loads it.

In this lesson we will split the program into two separate programs:
-   one focused on BPF/XDP loading (<xdp_loader.c>) and
-   one focused on reading and printing stats (<xdp_stats.c>).

The basic quest revolves around how to share or obtain the UNIX
file descriptor pointing to the BPF map from another program than the one
that created the map.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   Solutions to basic03 assignments (See section )
-   What you will learn in this lesson (See section )
    -   bpf syscall wrappers (See section )
    -   Mounting the BPF file system (See section )
    -   Gotchas with pinned maps (See section )
    -   Deleting maps on XDP reload (See section )
-   Assignments (See section )
    -   Assignment 1: (xdp\_stats.c) reload map file-descriptor (See section )
    -   Assignment 2: (xdp\_loader.c) reuse pinned map (See section )

# Solutions to basic03 assignments<a id="sec-2" name="sec-2"></a>

The assignments in [basic03](../basic03-map-counter) have been "solved" or implemented in this basic04
lesson. Thus, this functions as the reference solution for basic03.

# What you will learn in this lesson<a id="sec-3" name="sec-3"></a>

## bpf syscall wrappers<a id="sec-3-1" name="sec-3-1"></a>

When splitting up the [xdp\_load\_and\_stats.c](../basic03-map-counter/xdp_load_and_stats.c) program into <xdp_loader.c>
and <xdp_stats.c>, notice that xdp\_stats.c no longer includes
`#<bpf/libbpf.h>`. This is because xdp\_stats doesn't use any of the advanced
libbpf "object" related functions, it only uses the basis bpf syscall
wrappers, which libbpf also provides.

The bpf syscall wrappers are provided by libbpf via the `#<bpf/bpf.h>`
include file, which for this tutorial setup lives in
`../libbpf/src/root/usr/include/bpf/bpf.h` (but see also the [source bpf.h in
the libbpf github repository](https://github.com/libbpf/libbpf/blob/master/src/bpf.h)).

The point here is that libbpf keeps the low-level bpf-syscall wrappers in
separate files [bpf.h](https://github.com/libbpf/libbpf/blob/master/src/bpf.h) and [bpf.c](https://github.com/libbpf/libbpf/blob/master/src/bpf.c). Thus, we could shrink the size of our binary
by not linking with libbpf.a. However, for ease of use we just link
everything with the full library in this tutorial.

## Mounting the BPF file system<a id="sec-3-2" name="sec-3-2"></a>

The mechanism used for sharing BPF maps between programs is called
*pinning*. What this means is that we create a file for each map under a
special file system mounted at `/sys/fs/bpf/`. If this file system is not
mounted, our attempts to pin BPF objects will fail, so we need to make sure
it is mounted.

The needed mount command is:

    mount -t bpf bpf /sys/fs/bpf/

If you followed the tutorial you will likely already have gotten this
mounted without noticing. As both iproute2 'ip' and our [testenv](../testenv) will
automatically mount it to the default location under `/sys/fs/bpf/`.
If not, use the above command to mount it.

## Gotchas with pinned maps<a id="sec-3-3" name="sec-3-3"></a>

Pinning all maps in a `bpf_object` using libbpf is easily done with:
`bpf_object__pin_maps(bpf_object, pin_dir)`, which we use in `xdp_loader`.

To avoid filename collisions, we create a subdirectory named after the
interface that we are loading the BPF program onto. The libbpf
`bpf_object__pin_maps()` call even handles creating this subdirectory if it
doesn't exist.

However, if you open <xdp_loader.c> and look at our function
`pin_maps_in_bpf_object()`, you will see that due to corner cases things are
slightly more complicated. E.g., we also need to handle cleanup of previous
XDP programs that not have cleaned up their maps, which we choose to do via
`bpf_object__unpin_maps()`. If this is the first usage, then we should not
try to "unpin maps" as that will fail.

There is one corner case that we currently don't handle, namely the case
where our BPF-prog get extended with a new map, and is loaded as a
replacement for an existing BPF progam that doesn't contain this new map. In
this case, `bpf_object__unpin_maps()` won't finding the new map to unlink,
and therefore fail in its operation.

## Deleting maps on XDP reload<a id="sec-3-4" name="sec-3-4"></a>

When reloading the XDP BPF program via our `xdp_loader`, the existing pinned
maps are not reused. This differs from iproute2 tool BPF-loader (`ip` and
`tc` commands), which will reuse the existing pinned maps, rather than
creating new maps.

This is a design choice, mostly because libbpf doesn't have easy support for
this, but also because it is easier that the counters reset to zero to
observe if our program works. One can easily imagine that for real
applications it could be a problem that the counters reset to zero for the
different stats tools. Even for our split `xdp_stats` program it is
annoying, as you must remember to restart the `xdp_stats` tool, after
reloading via `xdp_loader`, else it will be watching the wrong FD.
(See Assignment 1 (See section ) for workaround)

### Reusing maps with libbpf<a id="sec-3-4-1" name="sec-3-4-1"></a>

The libbpf library can **reuse and replace** a map with an existing map file
descriptor, via the libbpf API call: `bpf_map__reuse_fd()`. But you cannot
use `bpf_prog_load()` for this; instead you have to code it yourself, as you
need a step in-between `bpf_object__open()` and `bpf_object__load`. The
basic steps needed looks like:

	# get file descrdescriptor of existed pinned map file system
    int pinned_map_fd = bpf_obj_get("/sys/fs/bpf/veth0/xdp_stats_map");

    struct bpf_object *obj = bpf_object__open(cfg.filename);
	# Get map object in new bpf object
    struct bpf_map    *map = bpf_object__find_map_by_name(obj, "xdp_stats_map");
	#set to reuse new map with existed file system.
    bpf_map__reuse_fd(map, pinned_map_fd);
    bpf_object__load(obj);


(Hint: see Assignment 2 (See section ))

# Assignments<a id="sec-4" name="sec-4"></a>

## Assignment 1: (xdp\_stats.c) reload map file-descriptor<a id="sec-4-1" name="sec-4-1"></a>

As mentioned above, the `xdp_stats` tool will not detect if `xdp_loader`
loads new maps and new BPF programs, and will need to be restarted. This is
annoying. The **assignment** is to reload the map file descriptor dynamically,
such that the `xdp_stats` program doesn't need to be restarted.

There are several solutions to this. The naive solution is to reopen the
pinned map file each time; but how do you detect that the file changed? If
you don't detect when dealing with a new map, then the stats diff between
two measurements will be negative. Think about solutions were you
remember/use the ID number to detect changes, either via the map ID or XDP
BPF program ID.

## Assignment 2: (xdp\_loader.c) reuse pinned map<a id="sec-4-2" name="sec-4-2"></a>

As mentioned above, libbpf can reuse and replace a map with an existing map,
it just requires writing your own `bpf_prog_load()` (or
`bpf_prog_load_xattr`).

The **assignment** is to check in [xdp\_loader](xdp_loader.c) if there already is a pinned
version of the map "xdp\_stats\_map" and use libbpf `bpf_map__reuse_fd()` API
to reuse it, instead of creating a new map.

# Conlusion:

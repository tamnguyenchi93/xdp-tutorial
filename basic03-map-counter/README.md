<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. Things you will learn in this lesson</a>
<ul>
<li><a href="#sec-2-1">2.1. Defining a map</a></li>
<li><a href="#sec-2-2">2.2. libbpf map ELF relocation</a></li>
<li><a href="#sec-2-3">2.3. bpf_object to bpf_map</a></li>
<li><a href="#sec-2-4">2.4. Reading map values from userspace</a></li>
</ul>
</li>
<li><a href="#sec-3">3. Assignments</a>
<ul>
<li><a href="#sec-3-1">3.1. Assignment 1: Add bytes counter</a>
<ul>
<li><a href="#sec-3-1-1">3.1.1. Assignment 1.1: Update the BPF program</a></li>
<li><a href="#sec-3-1-2">3.1.2. Assignment 1.2: Update the userspace program</a></li>
</ul>
</li>
<li><a href="#sec-3-2">3.2. Assignment 2: Handle other XDP actions stats</a></li>
<li><a href="#sec-3-3">3.3. Assignment 3: Per CPU stats</a></li>
</ul>
</li>
</ul>
</div>
</div>


In this lesson you will learn about BPF maps, the persistent storage
mechanism available to BPF programs. The assignments will give you hands-on
experience with extending the "value" size/content, and reading the contents
from userspace.

In this lesson we will only cover two simple maps types:
-   `BPF_MAP_TYPE_ARRAY` and
-   `BPF_MAP_TYPE_PERCPU_ARRAY`.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   Things you will learn in this lesson (See section )
    -   Defining a map (See section )
    -   libbpf map ELF relocation (See section )
    -   bpf\_object to bpf\_map (See section )
    -   Reading map values from userspace (See section )
-   Assignments (See section )
    -   Assignment 1: Add bytes counter (See section )
    -   Assignment 2: Handle other XDP actions stats (See section )
    -   Assignment 3: Per CPU stats (See section )

# Things you will learn in this lesson<a id="sec-2" name="sec-2"></a>

## Defining a map<a id="sec-2-1" name="sec-2-1"></a>

Creating a BPF map is done by defining a global struct `bpf_map_def` (in
<xdp_prog_kern.c>), with a special `SEC("maps")` as below:

    struct bpf_map_def SEC("maps") xdp_stats_map = {
            .type        = BPF_MAP_TYPE_ARRAY,
            .key_size    = sizeof(__u32),
            .value_size  = sizeof(struct datarec),
            .max_entries = XDP_ACTION_MAX,
    };

BPF maps are generic **key/value** stores (hence the `key_size` and
`value_size` parameters), with a given `type`, and maximum allowed entries
`max_entries`. Here we focus on the simple `BPF_MAP_TYPE_ARRAY`, which means
`max_entries` array elements get allocated when the map is first created.

The BPF map is accessible from both the BPF program (kernel) side and from
userspace. How this is done and how they differ is part of this lesson.

## libbpf map ELF relocation<a id="sec-2-2" name="sec-2-2"></a>

It is worth pointing out that everything goes through the bpf syscall. This
means that the user space program *must* create the maps and programs with
separate invocations of the bpf syscall. So how does a BPF program reference
a BPF map?

This happens by first loading all the BPF maps, and storing their
corresponding file descriptors (FDs). Then the ELF relocation table is used
to identify each reference  the BPF program makes to a given map; each such
reference is then rewritten, so the BPF byte code instructions use the right
map FD for each map.

All this needs to be done before the BPF program itself can be loaded into
the kernel. Fortunately, the libbpf library handles the ELF object decoding
and map reference relocation, transparently to the user space program
performing the loads.

## bpf\_object to bpf\_map<a id="sec-2-3" name="sec-2-3"></a>

As you learned in [basic02](../basic02-prog-by-name/) the libbpf API have "objects" and functions
working on/with these objects. The struct `bpf_object` represents the ELF
object itself (which is returned from our `load_bpf_and_xdp_attach()`
function).

Similarly to what we did for BPF functions, our load has a function called
`find_map_fd()` (in <xdp_load_and_stats.c>), which uses the library
function `bpf_object__find_map_by_name()` for finding the `bpf_map` object
with a given name. (Note, the length of the map name is provided by ELF and
is longer than what the name kernel stores, after loading it). After finding
the `bpf_object`, we obtain the map file descriptor via `bpf_map__fd()`.
There is also a libbpf function that wraps these two steps, which is called
`bpf_object__find_map_fd_by_name()`.

## Reading map values from userspace<a id="sec-2-4" name="sec-2-4"></a>

The contents of a map is read from userspace via the function
`bpf_map_lookup_elem()`, which is a simple syscall-wrapper, that operates on
the map file descriptor (FD). The syscall looks up the `key` and stores the
value into the memory area supplied by the value pointer. It is up to the
calling userspace program to ensure that the memory allocated to hold the
returned value is large enough to store the type of data contained in the
map. In our example we demonstrate how userspace can query the map FD and
get back some info in struct `bpf_map_info` via the syscall wrapper
`bpf_obj_get_info_by_fd()`.

For example, the program `xdp_load_and_stats` will periodically read the
xdp\_stats\_map value and produce some stats.

# Assignments<a id="sec-3" name="sec-3"></a>

The assignments are have "hint" marks in the code via `Assignment#num`
comments.

## Assignment 1: Add bytes counter<a id="sec-3-1" name="sec-3-1"></a>

The current assignment code only counts packets.  It is your **assignment** to
extend this to also count bytes.

Notice how the BPF map `xdp_stats_map` used:
-   `.value_size = sizeof(struct datarec)`

The BPF map has no knowledge about the data-structure used for the value
record, it only knows the size. (The [BPF Type Format](https://github.com/torvalds/linux/blob/master/Documentation/bpf/btf.rst) ([BTF](https://www.kernel.org/doc/html/latest/bpf/btf.html)) is an advanced
topic, that allows for associating data struct knowledge via debug info, but
we ignore that for now). Thus, it is up to the two sides (userspace and
BPF-prog kernel side) to ensure they stay in sync on the content and
structure of `value`. The hint here on the data structure used comes from
`sizeof(struct datarec)`, which indicate that `struct datarec` is used.

This `struct datarec` is defined in the include <common_kern_user.h> as:

    /* This is the data record stored in the map */
    struct datarec {
            __u64 rx_packets;
            /* Assignment#1: Add byte counters */
    };

### Assignment 1.1: Update the BPF program<a id="sec-3-1-1" name="sec-3-1-1"></a>

Next step is to update the kernel side BPF program: <xdp_prog_kern.c>.

To figure out the length of the packet, you need to learn about the context
variable `*ctx` with type [struct xdp\_md](https://elixir.bootlin.com/linux/v5.0/ident/xdp_md) that the BPF program gets a pointer
to when invoked by the kernel. This `struct xdp_md` is a little odd, as all
members have type `__u32`. However, this is not actually their real data
types, as access to this data-structure is remapped by the kernel when the
program is loaded into the kernel. Access gets remapped to struct `xdp_buff`
and also struct `xdp_rxq_info`.

    struct xdp_md {
            // (Note: type __u32 is NOT the real-type)
            __u32 data;
            __u32 data_end;
            __u32 data_meta;
            /* Below access go through struct xdp_rxq_info */
            __u32 ingress_ifindex; /* rxq->dev->ifindex */
            __u32 rx_queue_index;  /* rxq->queue_index */
    };

While we know this, the compiler doesn't. So we need to type-cast the fields
into void pointers before we can use them:

    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;

The next step is calculating the number of bytes in each packet, by simply
subtracting `data` from `data_end`, and update the datarec member.

    __u64 bytes = data_end - data; /* Calculate packet length */
    lock_xadd(&rec->rx_bytes, bytes);

### Assignment 1.2: Update the userspace program<a id="sec-3-1-2" name="sec-3-1-2"></a>

Now it is time to update the userspace program that reads stats (in
<xdp_load_and_stats.c>).

Update the functions:
-   `map_collect()` to also collect rx\_bytes.
-   `stats_print()` to also print rx\_bytes (adjust fmt string)

## Assignment 2: Handle other XDP actions stats<a id="sec-3-2" name="sec-3-2"></a>

Notice how the BPF map `xdp_stats_map` we defined above is actually an
array, with `max_entries=XDP_ACTION_MAX`. The idea with this is to keep
stats per [(enum) xdp\_action](https://elixir.bootlin.com/linux/latest/ident/xdp_action), but our program does not yet take advantage of
this.

The **assignment** is to extend userspace stats tool (in
<xdp_load_and_stats.c>) to collect and print these extra stats.

## Assignment 3: Per CPU stats<a id="sec-3-3" name="sec-3-3"></a>

Thus far, we have used atomic operations to increment our stats counters;
however, this is expensive as it inserts memory barriers to make sure
different CPUs don't garble each other's data. We can avoid this by using
another array type that stores its data in per-CPU storage. The drawback of
this is that we move the burden of summing to userspace.

To achieve this, the first step is to change map `type` (in
<xdp_prog_kern.c>) to use `BPF_MAP_TYPE_PERCPU_ARRAY`. If you only make
this change, the userspace program will detect this and complain, as we
query the map FD for some info (via `bpf_obj_get_info_by_fd()`) and e.g.
check the map type. Remember it is userspace's responsibility to make sure
the data record for the value is large enough.

Next step is writing a function that gets the values per CPU and sum these.
In the <xdp_load_and_stats.c>. You can copy paste this, and call it from
the switch-case statement in function `map_collect()`:

    /* BPF_MAP_TYPE_PERCPU_ARRAY */
    void map_get_value_percpu_array(int fd, __u32 key, struct datarec *value)
    {
            /* For percpu maps, userspace gets a value per possible CPU */
            unsigned int nr_cpus = bpf_num_possible_cpus();
            struct datarec values[nr_cpus];
            __u64 sum_bytes = 0;
            __u64 sum_pkts = 0;
            int i;
    
            if ((bpf_map_lookup_elem(fd, &key, values)) != 0) {
                    fprintf(stderr,
                            "ERR: bpf_map_lookup_elem failed key:0x%X\n", key);
                    return;
            }
    
            /* Sum values from each CPU */
            for (i = 0; i < nr_cpus; i++) {
                    sum_pkts  += values[i].rx_packets;
                    sum_bytes += values[i].rx_bytes;
            }
            value->rx_packets = sum_pkts;
            value->rx_bytes   = sum_bytes;
    }
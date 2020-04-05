<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. Using libbpf</a>
<ul>
<li><a href="#sec-2-1">2.1. Basic bpf_object usage</a></li>
<li><a href="#sec-2-2">2.2. Converting a bpf_object to bpf_program</a></li>
<li><a href="#sec-2-3">2.3. Hardware offloading</a></li>
</ul>
</li>
<li><a href="#sec-3">3. Assignments</a>
<ul>
<li><a href="#sec-3-1">3.1. Assignment 1: Setting up your test lab</a>
<ul>
<li><a href="#sec-3-1-1">3.1.1. A note about: The test environment and veth packets directions</a></li>
<li><a href="#sec-3-1-2">3.1.2. Recommended: Create an alias for testenv.sh</a></li>
<li><a href="#sec-3-1-3">3.1.3. Convenience: Skipping the environment name</a></li>
</ul>
</li>
<li><a href="#sec-3-2">3.2. Assignment 2: Add xdp_abort program</a></li>
</ul>
</li>
</ul>
</div>
</div>


In this lesson you will see that a BPF ELF file produced by LLVM can contain
more than one XDP program, and how you can select which one to load using
the **libbpf API**. Complete each of the assignments below to complete the
lesson.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   Using libbpf (See section )
    -   Basic bpf\_object usage (See section )
    -   Converting a bpf\_object to bpf\_program (See section )
    -   Hardware offloading (See section )
-   Assignments (See section )
    -   Assignment 1: Setting up your test lab (See section )
    -   Assignment 2: Add xdp\_abort program (See section )

# Using libbpf<a id="sec-2" name="sec-2"></a>

The libbpf API not only provides the basic system call wrappers (which are
defined in libbpf [bpf.h](https://github.com/libbpf/libbpf/blob/master/src/bpf.h)). The API also provides "[objects](https://github.com/libbpf/libbpf/blob/master/src/README.rst#objects)" and functions to
work with them (defined in include [libbpf.h](https://github.com/libbpf/libbpf/blob/master/src/libbpf.h)).

The C structs corresponding to the libbpf objects are:
-   struct `bpf_object`
-   struct `bpf_program`
-   struct `bpf_map`

These structs are for libbpf internal use, and you must use the API
functions to interact with the objects. Functions that work with an object
are named after the struct, followed by a double underscore, and a
descriptive name.

Lets look at a practical usage of `bpf_object` and `bpf_program`.

## Basic bpf\_object usage<a id="sec-2-1" name="sec-2-1"></a>

In <xdp_loader.c> the function `__load_bpf_object_file()` now returns a
libbpf struct `bpf_object` pointer (while in the basic01 lesson we just
returned the file descriptor to the first B PF prog).

The struct `bpf_object` represents an ELF object, which can be of various
types.

## Converting a bpf\_object to bpf\_program<a id="sec-2-2" name="sec-2-2"></a>

In <xdp_loader.c> the function `__load_bpf_and_xdp_attach()` uses
`bpf_object__find_program_by_title()` on the bpf\_object, which finds a
program by the "SEC" name (not the C-function name). The return type is a
struct `bpf_program` object, and we can use the function `bpf_program__fd()`
for getting the file descriptor that we want to attach to the XDP hook.

## Hardware offloading<a id="sec-2-3" name="sec-2-3"></a>

XDP can also be offloaded to run in the NIC hardware (on some
offload-capable NICs).

The XDP flag XDP\_FLAGS\_HW\_MODE enables (requests) hardware offloading, and
is set with the long-option `--offload-mode`. The [loader code](xdp_loader.c) also needs to
use a more advanced libbpf API call `bpf_prog_load_xattr()`, which allows us
to set the ifindex, as this is needed at load time to enable HW offloading.

There are some details on how to get hardware offloading working on
Netronome's Agilio SmartNICs in <xdp_offload_nfp.md>; for instance, the
firmware needs updating to support eBPF.

# Assignments<a id="sec-3" name="sec-3"></a>

## Assignment 1: Setting up your test lab<a id="sec-3-1" name="sec-3-1"></a>

As this lesson involves loading and selecting an XDP program that simply
drops all packets (via action `XDP_DROP`), you will need to load it on a
real interface to observe what is happening. To do this, we establish a test
lab environment. In the [testenv/](../testenv/) directory you will find a script
`testenv.sh` that helps you setup a test lab based on `veth` devices and
network namespaces.

E.g. run the script like:

    $ sudo ../testenv/testenv.sh setup --name veth-basic02
    Setting up new environment 'veth-basic02'
    Setup environment 'veth-basic02' with peer ip fc00:dead:cafe:1::2.

This results in the creation of an (outer) interface named: `veth-basic02`.
You can test that the environment network is operational by pinging the peer
IPv6 address `fc00:dead:cafe:1::2` (as seen in the script output).

The **assignment** is to manually load the compiled xdp program in the ELF OBJ
file `xdp_prog_kern.o`, using the `xdp_loader` program in this directory.
Observe the available options you can give the xdp\_loader via `--help`. Try
to select the program section named `xdp_drop` via `--progsec`, and observe
via ping that packets gets dropped.

Here are some example commands:

    sudo ./xdp_loader --help
    sudo ./xdp_loader --dev veth-basic02
    sudo ./xdp_loader --dev veth-basic02 --force --progsec xdp_drop
    sudo ./xdp_loader --dev veth-basic02 --force --progsec xdp_pass

The testenv script also has a helper command for "load" which will use the
`xdp_loader` program in the current directory:

    sudo ../testenv/testenv.sh load --name veth-basic02
    sudo ../testenv/testenv.sh load --name veth-basic02 -- --force
    sudo ../testenv/testenv.sh load --name veth-basic02 -- --force --progsec xdp_drop
    sudo ../testenv/testenv.sh load --name veth-basic02 -- --force --progsec xdp_pass

### A note about: The test environment and veth packets directions<a id="sec-3-1-1" name="sec-3-1-1"></a>

When you load an XDP program on the interface visible on your host machine,
it will operate on all packets arriving **to** that interface. And since
packets that are sent from one interface in a veth pair will arrive at the
other end, the packets that your XDP program will see are the ones sent from
**within** the network namespace (netns). This means that when you are
testing, you should do the ping from **within** the network namespace that
were created by the script.

You can "enter" the namespace manually (via `sudo ip netns exec veth-basic02
/bin/bash`) or via the script like:

    $ sudo ../testenv/testenv.sh enter --name veth-basic02
    # ping fc00:dead:cafe:1::1

To make this ping connectivity test easier, the script also has a `ping`
command that pings from within the netns:

    $ sudo ../testenv/testenv.sh ping --name veth-basic02

You should note that, the **cool thing** about using netns as a testlab is
that we can still "enter" the netns even-when XDP is dropping all packets.

### Recommended: Create an alias for testenv.sh<a id="sec-3-1-2" name="sec-3-1-2"></a>

To have faster access to the testenv.sh script, we recommend that you create
a shell alias (called `t`). The testenv script even has a command helper
for this purpose:

    $ ../testenv/testenv.sh alias
    Eval this with `eval $(../testenv/testenv.sh alias)` to create shell alias
    WARNING: Creating sudo alias; be careful, this script WILL execute arbitrary programs
    
    alias t='sudo /home/fedora/git/xdp-tutorial/testenv/testenv.sh'

As pointed out, run:

    eval $(../testenv/testenv.sh alias)

You should now be able to run testenv commands as `t <command>` (e.g., `t
ping`). All subsequent examples will use this syntax.

### Convenience: Skipping the environment name<a id="sec-3-1-3" name="sec-3-1-3"></a>

The testenv script will save the last used testenv name, so in most cases
you can skip the `--name` parameter when running the script. If you don't
specify a name when you run `t setup`, a random name will be generated for
you.

You can have several active test environments at the same time, and you can
always select a specific one using the `--name` parameter. Run `t status` to
see the currently selected environment (i.e., the one that will be used if
you don't specify one with `--name`), as well as the list of all currently
active environments.

## Assignment 2: Add xdp\_abort program<a id="sec-3-2" name="sec-3-2"></a>

Add a new program section "xdp\_abort" in <xdp_prog_kern.c> that uses
(returns) the XDP action `XDP_ABORTED` (and compile via `make`). Load this
new program, e.g. similar to above:

    sudo ./xdp_loader --dev veth-basic02 --force --progsec xdp_abort

**Lesson**: XDP\_ABORTED is different from XDP\_DROP, because it triggers the
tracepoint named `xdp:xdp_exception`.

While pinging from inside the namespace, record this tracepoint and observe
these records. E.g with perf like this:

    sudo perf record -a -e xdp:xdp_exception sleep 4
    sudo perf script
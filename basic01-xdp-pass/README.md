<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. First step: setup dependencies</a></li>
<li><a href="#sec-3">3. Compiling example code</a>
<ul>
<li><a href="#sec-3-1">3.1. Simple XDP code</a></li>
<li><a href="#sec-3-2">3.2. Compiling process</a></li>
<li><a href="#sec-3-3">3.3. Looking into the BPF-ELF object</a></li>
</ul>
</li>
<li><a href="#sec-4">4. Loading and the XDP hook</a>
<ul>
<li><a href="#sec-4-1">4.1. Loading via iproute2 ip</a></li>
<li><a href="#sec-4-2">4.2. Loading using xdp_pass_user</a></li>
</ul>
</li>
</ul>
</div>
</div>


Welcome to the first step in this XDP tutorial.

The programming language for XDP is eBPF (Extended Berkeley Packet Filter)
which we will just refer to as BPF. Thus, this tutorial will also be
relevant for learning how to write other BPF programs; however, the main
focus is on BPF programs that can be used in the XDP-hook. In this and the
following couple of lessons we will be focusing on the basics to get up and
running with BPF; the later lessons will then build on this to teach you how
to do packet processing with XDP.

Since this is the first lesson, we will start out softly by not actually
including any assignments. Instead, just read the text below and make sure
you can load the program and that you understand what is going on.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   First step: setup dependencies (See section )
-   Compiling example code (See section )
    -   Simple XDP code (See section )
    -   Compiling process (See section )
    -   Looking into the BPF-ELF object (See section )
-   Loading and the XDP hook (See section )
    -   Loading via iproute2 ip (See section )
    -   Loading using xdp\_pass\_user (See section )

# First step: setup dependencies<a id="sec-2" name="sec-2"></a>

There are a number of setup dependencies, that are needed in order to
compile the source code in this git repository. Please go read and complete
the <../setup_dependencies.md> guide if you haven't already.

Then return here, and see if the next step compiles.

# Compiling example code<a id="sec-3" name="sec-3"></a>

If you completed the setup dependencies guide, then you should be able to
simply run the `make` command, in this directory. (The [Makefile](Makefile) and
[common.mk](../common/common.mk) will try to be nice and detect if you didn't complete the setup
steps).

## Simple XDP code<a id="sec-3-1" name="sec-3-1"></a>

The very simple XDP code used in this step is located in
<xdp_pass_kern.c>, and displayed below:

    SEC("xdp")
    int  xdp_prog_simple(struct xdp_md *ctx)
    {
            return XDP_PASS;
    }

## Compiling process<a id="sec-3-2" name="sec-3-2"></a>

The LLVM+clang compiler turns this restricted-C code into BPF-byte-code and
stores it in an ELF object file, named `xdp_pass_kern.o`.

## Looking into the BPF-ELF object<a id="sec-3-3" name="sec-3-3"></a>

You can inspect the contents of the `xdp_pass_kern.o` file with different
tools like `readelf` or `llvm-objdump`. As the Makefile enables the debug
option `-g` (LLVM version >= 4.0), the llvm-objdump tool can annotate
assembler output with the original C code:

Run: `llvm-objdump -S xdp_pass_kern.o`

    xdp_pass_kern.o:        file format ELF64-BPF
    
    Disassembly of section xdp:
    xdp_prog_simple:
    ; {
           0:       b7 00 00 00 02 00 00 00         r0 = 2
    ; return XDP_PASS;
           1:       95 00 00 00 00 00 00 00         exit

If you don't want to see the raw BPF instructions add: `-no-show-raw-insn`.
The define/enum XDP\_PASS has a value of 2, as can be seen in the dump. The
section name "xdp" was defined by `SEC("xdp")`, and the `xdp_prog_simple:`
is our C-function name.

# Loading and the XDP hook<a id="sec-4" name="sec-4"></a>

As you should understand by now, the BPF byte code is stored in an ELF file.
To load this into the kernel, userspace needs an ELF loader to read the file
and pass it into the kernel in the right format. The **libbpf** library
provides both an ELF loader and several XDP helper functions. In this
tutorial you will learn how to write C code using this library, which is
where our libelf-devel dependency comes from.

The C code in <xdp_pass_user.c> (which gets compiled to the program
`xdp_pass_user`) shows how to write a BPF loader specifically for our
`xdp_pass_kern.o` ELF file. This loader attached the program in the ELF file
as an XDP hook on a network device.

## Loading via iproute2 ip<a id="sec-4-1" name="sec-4-1"></a>

It does seem overkill to write a C program to simply load and attach a
specific BPF-program. However, we still include this in the tutorial
since it will help you integrate BPF into other Open Source projects.

As an alternative to writing a new loader, the standard iproute2 tool also
contains a BPF ELF loader. However, this loader is not based on libbpf,
which unfortunately makes it incompatible when starting to use BPF maps.

The iproute2 loader can be used with the standard `ip` tool; so in this case
you can actually load our ELF-file `xdp_pass_kern.o` (where we named our
ELF section "xdp") like this:

    ip link set dev lo xdpgeneric obj xdp_pass_kern.o sec xdp

Listing the device via `ip link show` also shows the XDP info:

    $ ip link show dev lo
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 xdpgeneric qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        prog/xdp id 220 tag 3b185187f1855c4c jited

Removing the XDP program again from the device:

    ip link set dev lo xdpgeneric off

## Loading using xdp\_pass\_user<a id="sec-4-2" name="sec-4-2"></a>

To load the program using our own loader, simply issue this command:

    $ sudo ./xdp_pass_user --dev lo --skb-mode
    Success: Loading XDP prog name:xdp_prog_simple(id:225) on device:lo(ifindex:1)

Loading it again will fail, as there is already a program loaded. This is
because we use the xdp\_flag `XDP_FLAGS_UPDATE_IF_NOEXIST`. This is good
practice to avoid accidentally unloading an unrelated XDP program.

    $ sudo ./xdp_pass_user --dev lo --skb-mode
    ERR: dev:lo link set xdp fd failed (16): Device or resource busy
    Hint: XDP already loaded on device use --force to swap/replace

As the hint suggest, the option `--force` can be used to replace the
existing XDP program.

    $ sudo ./xdp_pass_user --dev lo --skb-mode --force
    Success: Loading XDP prog name:xdp_prog_simple(id:231) on device:lo(ifindex:1)

You can list XDP programs  on the device using different commands, and verify
that the program ID is the same:
-   `ip link list dev lo`
-   `bpftool net list dev lo`
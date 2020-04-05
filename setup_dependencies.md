<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. Based on libbpf</a>
<ul>
<li><a href="#sec-2-1">2.1. libbpf as git-submodule</a></li>
</ul>
</li>
<li><a href="#sec-3">3. Dependencies</a>
<ul>
<li><a href="#sec-3-1">3.1. Packages on Fedora</a></li>
<li><a href="#sec-3-2">3.2. Packages on Debian/Ubuntu</a></li>
<li><a href="#sec-3-3">3.3. Packages on openSUSE</a></li>
</ul>
</li>
<li><a href="#sec-4">4. Kernel headers dependency</a>
<ul>
<li><a href="#sec-4-1">4.1. Packages on Fedora</a></li>
<li><a href="#sec-4-2">4.2. Packages on Debian/Ubuntu</a></li>
<li><a href="#sec-4-3">4.3. Packages on openSUSE</a></li>
</ul>
</li>
<li><a href="#sec-5">5. Recommended tools</a>
<ul>
<li><a href="#sec-5-1">5.1. Packages on Fedora</a></li>
<li><a href="#sec-5-2">5.2. Packages on Ubuntu</a></li>
<li><a href="#sec-5-3">5.3. Packages on Debian</a></li>
<li><a href="#sec-5-4">5.4. Packages on openSUSE</a></li>
</ul>
</li>
</ul>
</div>
</div>


Before you can start completing the steps in this XDP-tutorial, go though
this document and install the needed software packages.

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   Based on libbpf (See section )
    -   libbpf as git-submodule (See section )
-   Dependencies (See section )
    -   Packages on Fedora (See section )
    -   Packages on Debian/Ubuntu (See section )
    -   Packages on openSUSE (See section )
-   Kernel headers dependency (See section )
    -   Packages on Fedora (See section )
    -   Packages on Debian/Ubuntu (See section )
    -   Packages on openSUSE (See section )
-   Recommended tools (See section )
    -   Packages on Fedora (See section )
    -   Packages on Debian/Ubuntu (See section )
    -   Packages on openSUSE (See section )

# Based on libbpf<a id="sec-2" name="sec-2"></a>

This XDP-tutorial leverages [libbpf](https://github.com/libbpf/libbpf/) to ease development and loading of
BPF-programs. The library libbpf is part of the kernel tree under
[tools/lib/bpf](https://github.com/torvalds/linux/blob/master/tools/lib/bpf/README.rst), but Facebook engineers maintain a stand-alone build on
GitHub under <https://github.com/libbpf/libbpf>.

## libbpf as git-submodule<a id="sec-2-1" name="sec-2-1"></a>

This repository uses [libbpf](https://github.com/libbpf/libbpf) as a git-submodule. After cloning this repository you need to run the command:

    git submodule update --init

If you want submodules to be part of the clone, you can use this command:

    git clone --recurse-submodules https://github.com/xdp-project/xdp-tutorial

If you need to add this to your own project, you can use the command:

    git submodule add https://github.com/libbpf/libbpf/ libbpf

# Dependencies<a id="sec-3" name="sec-3"></a>

The main dependencies are `libbpf`, `llvm`, `clang` and `libelf`. LLVM+clang
compiles our restricted-C programs into BPF-byte-code, which is stored in an
ELF object file (`libelf`), that is loaded by `libbpf` into the kernel via
the `bpf` syscall. Some of the lessons also use the `perf` utility to
track the kernel behaviour through tracepoints.

The Makefiles in this repo will try to detect if you are missing some
dependencies, and give you some pointers.

## Packages on Fedora<a id="sec-3-1" name="sec-3-1"></a>

On a machine running the Fedora Linux distribution, install the packages:

    $ sudo dnf install clang llvm
    $ sudo dnf install elfutils-libelf-devel perf

Note also that Fedora by default sets a limit on the amount of locked memory
the kernel will allow, which can interfere with loading BPF maps. The
`testenv.sh` script will adjust this for you, but if you're not using that
you will probably run into problems. Use this command to raise the limit:

    # ulimit -l 1024

Note that you need to do this in the shell you are using to load programs
(in particular, it won't work with `sudo`).

## Packages on Debian/Ubuntu<a id="sec-3-2" name="sec-3-2"></a>

On Debian and Ubuntu installations, install the dependencies like this:

    $ sudo apt install clang llvm libelf-dev gcc-multilib

To install the 'perf' utility, run this on Debian:

    $ sudo apt install linux-perf

or this on Ubuntu:

    $ sudo apt install linux-tools-$(uname -r)

## Packages on openSUSE<a id="sec-3-3" name="sec-3-3"></a>

On a machine running the openSUSE distribution, install the packages:

    $ sudo zypper install clang llvm libelf-devel perf

# Kernel headers dependency<a id="sec-4" name="sec-4"></a>

The Linux kernel provides a number of header files, which are usually installed
in `/usr/include/linux`. The different Linux distributions usually provide a
software package with these headers.

Some of the header files (we depend on) are located in the kernel tree under
include/uapi/linux/ (e.g. include/uapi/linux/bpf.h), but you should not include
those files as they go through a conversion process when exported/installed into
distros' `/usr/include/linux` directory. In the kernel git tree you can run the
command: `make headers_install` which will create a lot of headers files in
directory "usr/".

For now, this tutorial depends on kernel headers package provided by your
distro. We may choose to shadow some of these later.

## Packages on Fedora<a id="sec-4-1" name="sec-4-1"></a>

On a machine running the Fedora Linux distribution, install the package:

    $ sudo dnf install kernel-headers

## Packages on Debian/Ubuntu<a id="sec-4-2" name="sec-4-2"></a>

On Debian and Ubuntu installations, install the headers like this

    $ sudo apt install linux-headers-$(uname -r)

## Packages on openSUSE<a id="sec-4-3" name="sec-4-3"></a>

On a machine running the openSUSE distribution, install the package:

    $ sudo zypper install kernel-devel

# Recommended tools<a id="sec-5" name="sec-5"></a>

The `bpftool` is the recommended tool for inspecting BPF programs running on
your system. It also offers simple manipulation of eBPF programs and maps.
The `bpftool` is part of the Linux kernel tree under [tools/bpf/bpftool/](https://github.com/torvalds/linux/tree/master/tools/bpf/bpftool), but
some Linux distributions also ship the tool as a software package.

## Packages on Fedora<a id="sec-5-1" name="sec-5-1"></a>

On a machine running the Fedora Linux distribution, install package:

    $ sudo dnf install bpftool

## Packages on Ubuntu<a id="sec-5-2" name="sec-5-2"></a>

Starting from Ubuntu 19.10, bpftool can be installed with:

    $ sudo apt install linux-tools-common linux-tools-generic

(Ubuntu 18.04 LTS also has it, but it is an old and quite limited bpftool
version.)

## Packages on Debian<a id="sec-5-3" name="sec-5-3"></a>

Unfortunately, bpftool is not officially packaged for Debian
[yet](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=896165)).

However, note that an unofficial
[.deb package](https://help.netronome.com/helpdesk/attachments/36025601060)
is provided by Netronome
[on their support website](https://help.netronome.com/support/solutions/articles/36000050009-agilio-ebpf-2-0-6-extended-berkeley-packet-filter).
The binary is statically linked, and should work on any x86-64 Linux machine.

## Packages on openSUSE<a id="sec-5-4" name="sec-5-4"></a>

On a machine running the openSUSE Tumbleweed distribution, install package:

    $ sudo zypper install bpftool
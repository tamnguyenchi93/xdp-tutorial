<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Table of Contents&#xa0;&#xa0;&#xa0;<span class="tag"><span class="TOC">TOC</span></span></a></li>
<li><a href="#sec-2">2. Introduction</a></li>
<li><a href="#sec-3">3. First step: Setup dependencies</a></li>
<li><a href="#sec-4">4. How the lessons are organised</a>
<ul>
<li><a href="#sec-4-1">4.1. Basic setup lessons</a></li>
<li><a href="#sec-4-2">4.2. Packet processing lessons</a></li>
<li><a href="#sec-4-3">4.3. Advanced lessons</a></li>
</ul>
</li>
</ul>
</div>
</div>


This repository contains a tutorial that aims to introduce you to the basic
steps needed to effectively write programs for the eXpress Data Path (XDP)
system in the Linux kernel, which offers high-performance programmable
packet processing integrated with the kernel.

The tutorial is composed of a number of lessons, each of which has its own
repository. Start with the lessons starting with "basicXX", and read the
README.org file in each repository for instructions for that lesson.

Keep reading below for an introduction to XDP and an overview of what you
will learn in this tutorial, or jump [straight to the first lesson](basic01-xdp-pass/README.md).

# Table of Contents     :TOC:<a id="sec-1" name="sec-1"></a>

-   Introduction (See section )
-   First step: Setup dependencies (See section )
-   How the lessons are organised (See section )
    -   Basic setup lessons (See section )
    -   Packet processing lessons (See section )
    -   Advanced lessons (See section )

# Introduction<a id="sec-2" name="sec-2"></a>

XDP is a part of the upstream Linux kernel, and enables users to install
packet processing programs into the kernel, that will be executed for each
arriving packet, before the kernel does any other processing on the data.
The programs are written in restricted C, and compiled into the eBPF byte
code format that is executed and JIT-compiled in the kernel, after being
verified for safety. This approach offers great flexibility and high
performance, and integrates well with the rest of the system. For a general
introduction to XDP, read [the academic paper (pdf)](https://github.com/xdp-project/xdp-paper/blob/master/xdp-the-express-data-path.pdf), or the [Cilium BPF
reference guide](https://cilium.readthedocs.io/en/latest/bpf/).

This tutorial aims to be a practical introduction to the different steps
needed to successfully write useful programs using the XDP system. We assume
you have a basic understanding of Linux networking and how to configure it
with the `iproute2` suite of tools, but assume no prior experience with eBPF
or XDP.

The tutorial is a work in progress, and will be trial-run at the [Netdev
Conference](https://www.netdevconf.org/0x13/session.html?tutorial-XDP-hands-on) in Prague in March 2019. However, we intend for it to (eventually)
be self-contained and something that anyone can go through to learn the XDP
basics. Input and contributions to advance towards this goal are very
welcome; just open issues or pull requests in the [Github repository](https://github.com/xdp-project/xdp-tutorial/).

# First step: Setup dependencies<a id="sec-3" name="sec-3"></a>

Before you can start completing step in this tutorial, you will need to
install a few dependencies on your system. These are described in
<setup_dependencies.md>.

We also provide a helper script that will set up a test environment with
virtual interfaces for you to test your code on. This is introduced in the
basic lessons, and also has [it's own README file](testenv/README.md).

# How the lessons are organised<a id="sec-4" name="sec-4"></a>

The tutorial is organised into a number of lessons; each lesson has its own
subdirectory, and the lessons are grouped by category:

-   Basic setup (directories starting with basicXX)
-   Packet processing (directories starting with packetXX)
-   Advanced topics (directories starting with advancedXX)

We recommend you start with the "basic" lessons, and follow the lessons in
each category in numerical order. Read the README.org file in each lesson
directory for instructions on how to complete the lesson.

## Basic setup lessons<a id="sec-4-1" name="sec-4-1"></a>

We recommend you start with these lessons, as they will teach you how to
compile and inspect the eBPF programs that will implement your packet
processing code, how to load them into the kernel, and how to inspect the
state afterwards. As part of the basic lessons you will also be writing an
eBPF program loader that you will need in subsequent lessons.

## Packet processing lessons<a id="sec-4-2" name="sec-4-2"></a>

Once you have the basics figured out and know how to load programs into the
kernel, you are ready to start processing some packets. The lessons in the
packet progressing category will teach you about the different steps needed
to process data packets, including parsing, rewriting, instructing the
kernel about what to do with the packet after processing, and how to use
helpers to access existing kernel functionality.

## Advanced lessons<a id="sec-4-3" name="sec-4-3"></a>

After having completed the lessons in the basic and packet processing
categories, you should be all set to write your first real XDP program that
will do useful processing of the packets coming into the system. However,
there are some slightly more advanced topics that will probably be useful
once you start expanding your program to do more things.

The topics covered in the advanced lessons include how to make eBPF programs
in other parts of the kernel interact with your XDP program, passing
metadata between programs, best practices for interacting with userspace and
kernel features, and how to run multiple XDP programs on a single interface.
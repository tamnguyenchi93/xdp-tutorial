<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. SElinux workaround</a></li>
</ul>
</div>
</div>


This files contains notes about needed setup workarounds.

# SElinux workaround<a id="sec-1" name="sec-1"></a>

In current Fedora 29, SElinux deny bpftool access to listing maps via e.g.
commands:

    # bpftool map
    # bpftool map list

Users see this error:

    $ sudo bpftool map
    Error: can't get map by id (13): Permission denied

Other part of the bpftool command do work, like listing BPF-prog running:

    # bpftool prog
    # bpftool prog list

Filed: Red Hat [Bug 1688668](https://bugzilla.redhat.com/show_bug.cgi?id=1688668) - SElinux conflict with bpftool map listing
-   As of this writing it is already resolved
-   But not fully rolled out, so use below workaround
-   Fixed in selinux-policy version 3.14.2-51.fc29

Using the proposed workaround:
-   <https://bodhi.fedoraproject.org/updates/FEDORA-2019-4cc36fafbb>
-   sudo dnf upgrade &#x2013;enablerepo=updates-testing &#x2013;advisory=FEDORA-2019-4cc36fafbb
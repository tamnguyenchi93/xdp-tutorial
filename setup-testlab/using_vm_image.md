<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Host-OS dependencies</a>
<ul>
<li><a href="#sec-1-1">1.1. Fedora: libvirt software setup</a></li>
</ul>
</li>
<li><a href="#sec-2">2. Import/use VM-image</a>
<ul>
<li><a href="#sec-2-1">2.1. Use via virt-manager</a></li>
<li><a href="#sec-2-2">2.2. Use via virt-install</a></li>
</ul>
</li>
<li><a href="#sec-3">3. Default username+password</a></li>
</ul>
</div>
</div>


How can you use the provided VM-image (which were created as described in
<create_vm_image.md>).

# Host-OS dependencies<a id="sec-1" name="sec-1"></a>

First of all, the host-OS (likely your laptop) need some software packages for
running a virtual machine (VM) image.

## Fedora: libvirt software setup<a id="sec-1-1" name="sec-1-1"></a>

There is a guide for Fedora here:
-   <https://docs.fedoraproject.org/en-US/quick-docs/getting-started-with-virtualization/>

Fedora have a package collection called @virtualization.

    $ dnf groupinfo virtualization
    Group: Virtualization
     Description: These packages provide a graphical virtualization environment.
     Mandatory Packages:
       virt-install
     Default Packages:
       libvirt-daemon-config-network
       libvirt-daemon-kvm
       qemu-kvm
       virt-manager
       virt-viewer
     Optional Packages:
       guestfs-browser
       libguestfs-tools
       python3-libguestfs
       virt-top

Follow the instruction in guide link:

    sudo dnf group install --with-optional virtualization
    
    # After the packages install, start the libvirtd service:
    sudo systemctl start libvirtd
    
    # To start the service on boot, run:
    sudo systemctl enable libvirtd
    
    # I had to restart libvirtd
    sudo systemctl restart libvirtd
    
    # verify that the KVM kernel modules are properly loaded
    lsmod | grep kvm

# Import/use VM-image<a id="sec-2" name="sec-2"></a>

There are a number of ways to use/import the provided image.

## Use via virt-manager<a id="sec-2-1" name="sec-2-1"></a>

Create a new virtual machine and import provided disk image virt-manager
interface selecting "Import existing disk image" and adding CDROM drive
manually.

Use graphical tool: virt-manager
-   (If not already connected: connect to QEMU/KVM on localhost)
-   File -> "New Virtual Machine"
-   Radio-button: "Import existing disk image"
-   "Browse&#x2026;" for file:
    -   Select "F29-xdp-tutorial.qcow2" (Choose Volume)
-   Choose the operating system; name: Fedora 29
    -   Select "Forward"
-   Choose Memory and CPU settings
-   Choose: Name: "F29-xdp-tutorial"

## Use via virt-install<a id="sec-2-2" name="sec-2-2"></a>

You can create a new libvirt machine, that use this image, from the command
line using `virt-install`:

Here we assume you installed the VM image in:
-   /var/lib/libvirt/images/F29-xdp-tutorial.qcow2

    sudo virt-install --name F29-xdp-tutorial \
    --description 'Fedora 29 - XDP-tutorial' \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/F29-xdp-tutorial.qcow2 \
    --cdrom /dev/null \
    --os-type linux \
    --os-variant fedora29 \
    --network bridge=virbr0 \
    --graphics vnc,listen=127.0.0.1,port=5901 \
    --noautoconsole

Guess you don't prefer the graphical tool virt-manager.  You can start a
console login via:

    sudo virsh console F29-xdp-tutorial

You should login with user "fedora", observe the IP-address (e.g. ifconfig)
and then instead use SSH to login.  To exit the console use: `Ctrl + 5`.

# Default username+password<a id="sec-3" name="sec-3"></a>

The default username+password for your new VM image is:
-   Username: fedora
-   Password: xdptut

You should login and add your own SSH-key to the user "fedora"
authorized\_keys, e.g. via copy-paste into:

    cat >> /home/fedora/.ssh/authorized_keys
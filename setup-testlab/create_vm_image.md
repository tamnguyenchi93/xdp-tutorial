<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Installing needed packages for XDP-tutorial</a></li>
<li><a href="#sec-2">2. Downloading images</a>
<ul>
<li><a href="#sec-2-1">2.1. Initial setup for cloud-init</a>
<ul>
<li><a href="#sec-2-1-1">2.1.1. Create: cloud-init.iso</a></li>
</ul>
</li>
</ul>
</li>
<li><a href="#sec-3">3. Initial: Import/use VM-image</a>
<ul>
<li><a href="#sec-3-1">3.1. Create via virt-install</a></li>
<li><a href="#sec-3-2">3.2. Default user+password</a></li>
</ul>
</li>
<li><a href="#sec-4">4. Disk capacity</a>
<ul>
<li><a href="#sec-4-1">4.1. Increasing VM disk capacity</a></li>
<li><a href="#sec-4-2">4.2. Reduce VM image size</a></li>
</ul>
</li>
<li><a href="#sec-5">5. Extra manual changes</a>
<ul>
<li><a href="#sec-5-1">5.1. stow</a></li>
<li><a href="#sec-5-2">5.2. pahole</a></li>
<li><a href="#sec-5-3">5.3. Kernel samples/bpf/</a></li>
</ul>
</li>
</ul>
</div>
</div>


This document describe how the virtual machine (VM) image for the
XDP-tutorial were created.

# Installing needed packages for XDP-tutorial<a id="sec-1" name="sec-1"></a>

The Ansible setup in directory [ansible/](ansible) is used to define and install the
needed software packages.  This were used on the image after below setup.

# Downloading images<a id="sec-2" name="sec-2"></a>

Here we download a predefined "cloud" image, and update it to suit the
tutorial.

Finding some alternative images to download here:
-   <https://alt.fedoraproject.org/>
-   <https://alt.fedoraproject.org/cloud/>

Specifically downlaod the: "Cloud Base qcow2 image" (size 294M)
-   <https://download.fedoraproject.org/pub/fedora/linux/releases/29/Cloud/x86_64/images/Fedora-Cloud-Base-29-1.2.x86_64.qcow2>

Place a copy in: *var/lib/libvirt/images*
-   with filename: F29-xdp-tutorial.qcow2.

## Initial setup for cloud-init<a id="sec-2-1" name="sec-2-1"></a>

Use of the cloud image is annoying, because it requires creating a special
CDROM image to reconfigure. After reboots it still want to do some
cloud-init, which either requires mounting below CDROM-image or [disabling
cloud-init](https://fatmin.com/2017/10/19/how-to-disable-cloud-init-in-a-rhel-cloud-image/) via cmdline: touch /etc/cloud/cloud-init.disabled

### Create: cloud-init.iso<a id="sec-2-1-1" name="sec-2-1-1"></a>

As describe [here](https://www.technovelty.org/linux/running-cloud-images-locally.html), these cloud images need/tries to fetch some setup info. We
need to supply this, else you cannot login to them. This is supplied by
creating a special crafted ISO image.

Create two files:

    cat > meta-data << EOF
    instance-id: iid-local01
    local-hostname: xdp-tutorial
    EOF

    cat > user-data << EOF
    #cloud-config
    password: xdptut
    ssh_pwauth: True
    chpasswd: { expire: False }
    
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAEAQC+4vvIvwdZDBqVpeTHNv1QVxmWyK9rPTcIeAEssPly9aMe3Z0pzCikKnmH0biQQBn+hY3N7M6lrtE/n5znyClblS7k4Wud2GsZDjwmEYHPsi2/mf8JmJvNkJXTwd/1fOqr4LX4XFRVxbpT4cJ4qmtVSUjzc5I3a0/GJzm/0t9eXFHlIA/Ei+mFWF9b6y+0hWudb3Uwe1AwY1orM8imiHkS6/kztvD1FRJWZswwZi2fb5EfCCfyZCDGNDljueIWGt/I64iMgVBWuttyUxHvKkOd2AZ4up3K2JDnI0RC6pPWW4QiP6DLYYSRX9NofnO2/PaBjuKo1dGyGb8EAM5NrR/UxWAKpVThx8ya/j375ut2OsleCIhoj4Kb1depuZtGdJyj7p6c86K7pum4CqS8ar8XbnGsW1Nt6SaCOdcP4LkZAk/eGVMuRkFbU9C2fDaaipuzlh+kZxWS5/1PZVKfOK+X0xU8c0aBifqhy4kYpsLRkVPwv/EdjYbiKYdRZIJMIRgkyZDSStI2mJAzbIO4VVgGVWeRiVZsPw3nbt6GvgNCd1WGnjvR5LuvrFevDaXgnpsLHU42bwuuq30M+lKh4Ysu0xgniCZsEc7JWZkjzZi7s3I2r6Q2iL6hq+WheDsGEcrOFWo3FDe8mGDkmyGC3SzSUZmhJPav0WfWbWtPnz9a9Tmd48un7fngxYO9lQVBTotJ6uHs9JsrRENWDuSiIUQk9D2XB3x82tJiL3Pb+8CGaJqCzt5Zs0HkfF9K2LMw40ENcWwDMqdXmuHprUuwhTXk1tZtkQqAbghQVx+ivBmaq4issgPfyaUyCnETFjmmNXRirYAsWQUiYmxbz78jUBM7+idEDhvJANCprcgU8W8g8nvv9Gfbg781jCxZim5qm/3PpkSJGfcoQQYQyVguI1a9A2YrtK5EwHNFmVdW0v8Qs9gP60WGOI+2GIk73ePMsD99eowAKMBKAYbcst+sGGY3h8TS3md4TFT3zlgU4EgAOs3NDytxKJWI0G1I7FVVfbWRgIedNxT5Z0g9Mbw+rDo42lIEL6EBe3q1vkwg0bS7Y0+9HCMKDO3NFDKXhE9b/IYjB2QYb18ZlhxENFZPyXkTKKL5S9O7i5vTQ/efGZRNw/MVB52V6UPh+mY3KIUOkTyafUtlx0ufqshkJklTndRE9oOt/0iuLt1aM6ruK77HYY6gEvnHUZwJXsw4YbiaX+8m9RG7G/otxBihJZQTKO4LnB8XRZj7eZatV4MoXAxXuWMYvsB6Hlzg5J8gWP6egfk2dYoC9SAEko1LpAspMilJDEFtZZepYkMqH8wsFyXgRb5CUrSOh4jSRtBix/uMCPtHq+z77mzKCEZTy3L+Wzbr netoptimizer@github
    EOF

Create the ISO file that the images can use as CDROM drive:

    genisoimage -output cloud-init.iso -volid cidata -joliet -rock user-data meta-data

Then copy init.iso into /var/lib/libvirtd/images as well (if you already
have an non-functional cloud-image, you can connect this images as
virtual-CDROM drive).

    sudo cp cloud-init.iso /var/lib/libvirt/images/

# Initial: Import/use VM-image<a id="sec-3" name="sec-3"></a>

## Create via virt-install<a id="sec-3-1" name="sec-3-1"></a>

You can create a new libvirt machine, that use this image, from the command
line using `virt-install`.

Notice these two files must exist:
-   /var/lib/libvirt/images/F29-xdp-tutorial.qcow2
-   /var/lib/libvirt/images/cloud-init.iso

Use virt-install:

    sudo virt-install --name F29-xdp-tutorial \
    --description 'Fedora 29 - XDP-tutorial' \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/F29-xdp-tutorial.qcow2 \
    --cdrom  /var/lib/libvirt/images/cloud-init.iso \
    --os-type linux \
    --os-variant fedora29 \
    --network bridge=virbr0 \
    --graphics vnc,listen=127.0.0.1,port=5901 \
    --noautoconsole

Another important detail: If the machines doesn't give you a console after
rebooting, then you likely need to (again) add the CDROM drive with the
cloud-init.iso (/var/lib/libvirt/images/cloud-init.iso). This can be done
via virt-manager. You can also disable "cloud-init" via command line:
`sudo touch /etc/cloud/cloud-init.disabled`

## Default user+password<a id="sec-3-2" name="sec-3-2"></a>

End-result: A virtual machine with:
-   Username: fedora
-   Password: xdptut

This Fedora-Cloud image have a 10 sec. delay when logging in as root, remove
that via editing *root*.ssh/authorized\_keys.

# Disk capacity<a id="sec-4" name="sec-4"></a>

## Increasing VM disk capacity<a id="sec-4-1" name="sec-4-1"></a>

The remaining disc capacity is getting low, it will be unfortunate if this
disrupt the tutorial. Thus, we resize the VM disk. First shutdown the VM.

Following [this blogpost](https://nullr0ute.com/2018/08/increasing-a-libvirt-kvm-virtual-machine-disk-capacity/) we resize disk via commands.

On host-OS machine:

    cd /var/lib/libvirt/images/
    qemu-img resize F29-xdp-tutorial.qcow2 +2G
    qemu-img info   F29-xdp-tutorial.qcow2

Now power up the VM, login as root (or use sudo) for next commands on VM:

    echo ", +" | sfdisk -N 1 /dev/vda --no-reread
    partprobe
    resize2fs /dev/vda1

## Reduce VM image size<a id="sec-4-2" name="sec-4-2"></a>

Before uploading, use the tool `virt-sparsify` to reduce disk capacity used
by VM image. And afterwards also compress image, to reduce download size.

# Extra manual changes<a id="sec-5" name="sec-5"></a>

Cloned some git trees in user 'fedora' home directory.

List of git trees:
-   <https://github.com/xdp-project/xdp-tutorial/>
-   <https://github.com/netoptimizer/network-testing>
-   git://git.kernel.org/pub/scm/devel/pahole/pahole.git

## stow<a id="sec-5-1" name="sec-5-1"></a>

Use stow to easier keep track of manually installed packages under:
`/usr/local/`

Some manual setup (TODO move this to ansible):

<div class="export">
sudo mkdir *usr/local/stow*
sudo chown fedora *usr/local/stow*
sudo chgrp -R adm  *usr/local*
sudo chmod -R g+ws *usr/local*

</div>

Note, ansible-setup have already added /usr/local/lib to ldconfig.

## pahole<a id="sec-5-2" name="sec-5-2"></a>

Until LLVM 8.0.0 gets released, we can use pahole to create the BTF-info
into in the BPF-ELF files. The on Fedora 29 the package 'dwarves' that
contain pahole is too old to have the needed BTF-features (avail in version
1.12). Thus, we need to build it ourselves.

Building pahole:

    cd ~/git/pahole
    mkdir build
    cd build
    cmake -DCMAKE_INSTALL_PREFIX=/usr/local/stow/pahole-git01 -D__LIB=lib ..
    make

Stow part of the setup:

    cd /usr/local/stow/
    stow pahole-git01
    sudo ldconfig

## Kernel samples/bpf/<a id="sec-5-3" name="sec-5-3"></a>

Uploaded a compiled version of kernel samples/bpf/ directory.
The `xdp_monitor` tool could come in handy for participants.
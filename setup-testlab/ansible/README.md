<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Running</a></li>
<li><a href="#sec-2">2. Bootstrapping Fedora</a></li>
</ul>
</div>
</div>


This directory contains an Ansible setup, that installs the needed software
package dependencies for the XDP-tutorial.  It have been used on the VM
image that participants are provided.

# Running<a id="sec-1" name="sec-1"></a>

To run this ansible setup on your own testlab VM, edit the <hosts> and
update it with the correct VM IP-address. Verify that you can SSH login to
the VM with username: `fedora` and your SSH-key.

The script [run-on-hosts.sh](run-on-hosts.sh) can be used to run:
-   ansible-playbook -i hosts site.yml

# Bootstrapping Fedora<a id="sec-2" name="sec-2"></a>

Notice: This trick is ONLY needed first time, on an clean/fresh (VM) image.

For some reason `/usr/bin/python` were not installed in Fedora 29, which
Ansible complains about like this:

    $ ansible -i hosts --user=root -m ping all
    192.168.122.98 | FAILED! => {
        "changed": false,
        "module_stderr": "Shared connection to 192.168.122.98 closed.\r\n",
        "module_stdout": "/bin/sh: /usr/bin/python: No such file or directory\r\n",
        "msg": "The module failed to execute correctly, you probably need to set the interpreter.\nSee stdout/stderr for the exact error",
        "rc": 127
    }

The packages python and python-dnf needs to be installed. We have added a
<bootstrap-ansible.yml> that perform this via ansible, and it need to be
run like:

    ansible-playbook -i hosts --user=root bootstrap-ansible.yml

Afterwards we can test if the user `fedora` can run ansible:

    $ ansible -i hosts --user fedora -m ping all
    192.168.122.98 | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
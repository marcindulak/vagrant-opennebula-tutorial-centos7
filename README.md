-----------
Description
-----------

Configures OpenNebula cluster using libvirt/kvm nested virtualization in Vagrant.
Vagrant starts a CentOS 7 frontend and OpenNebula worker nodes. The OpenNebula frontend
is then used to manage ttylinux virtual machines on the OpenNebula worker nodes.

Tested on Ubuntu 14.04 host.


----------------------
Configuration overview
----------------------

The private 192.168.10.0/24 network is associated with the Vagrant eth1 interfaces.
The bridges br1 use those eth1 interfaces. eth0 is used by Vagrant.
Vagrant reserves eth0 and this cannot be currently changed
(see https://github.com/mitchellh/vagrant/issues/2093).

                 -------------------------------------------
                 |               Vagrant HOST              |
                 -------------------------------------------
                  |     |          |     |          |     |
              eth1| eth0|      eth1| eth0|      eth1| eth0|
                  |     |          |     |          |     |
                 ------------     ---------        ---------
                 | frontend |     | node1 |   ...  | nodeN |
                 ------------     ---------        ---------
                  |                |                |
         br1(eth1)|       br1(eth1)|       br1(eth1)|
      192.168.10.5|   192.168.10.11|   192.168.10.1N|
                  |                |                |
                  |               ------------------------------
                  |-------------- | cluster: node1, ..., nodeN |
                                  ------------------------------
                                   |
                               eth0|
                     192.168.10.100|
                                  -------
                                  | VM0 | ...
                                  -------


------------
Sample Usage
------------

Install Vagrant https://www.vagrantup.com/downloads.html

Make sure nested virtualization is enabled on your host (e.g. laptop)::

        $ modinfo kvm_intel | grep -i nested

and install, configure libvirt, as a **standard** user::

        $ sudo apt-get -y install kvm libvirt-bin
        $ sudo adduser $USER libvirtd  # logout and login again

Install https://github.com/pradels/vagrant-libvirt plugin dependencies::

        $ sudo apt-get install -y ruby-libvirt
        $ sudo apt-get install -y libxslt-dev libxml2-dev libvirt-dev zlib1g-dev

Disable the default libvirt network::

        $ virsh net-autostart default --disable
        $ sudo service libvirt-bin restart

**Note** libvirt may have problems if ip6tables are not running.
Make also sure that no other virtualization (VirtualBox, etc.)
is active on the host machine.

Start the virtual OpenNebula setup with::

        $ git clone https://github.com/marcindulak/vagrant-opennebula-centos7.git
        $ cd vagrant-opennebula-centos7
        $ vagrant plugin install vagrant-libvirt
        $ vagrant up --no-parallel

The above setup follows loosely the instructions from
http://docs.opennebula.org/4.14/design_and_installation/quick_starts/qs_centos7_kvm.html

After having the frontend and node running test a basic OpenNebula usage scenario::

- add the started worker nodes to the cluster:

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onehost create node1 -i kvm -v kvm -n dummy'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onehost list'"

- create a network template, consisting of potentially three virtual machines (SIZE = 3):

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'echo NAME = private > mynetwork.one; echo BRIDGE = br1 >> mynetwork.one; echo AR = [TYPE = IP4, IP = 192.168.10.100, SIZE = 3] >> mynetwork.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevnet list'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevnet create mynetwork.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevnet list'"

- fetch a ttylinux image from OpenNebula's marketplace::

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'oneimage create --name ttylinux --path http://marketplace.c12g.com/appliance/4fc76a938fb81d3517000003/download -d default'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'oneimage list'"

- create a VM template using that image::

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate create --name ttylinux --cpu 1 --vcpu 1 --memory 127 --arch x86_64 --disk ttylinux --nic private --vnc --ssh --net_context'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate list'"

- start the VM using this template::

            $ sleep 30  # wait for template to become ready
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate instantiate ttylinux'"
            $ sleep 300  # wait for the VM to start
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevm list'"
            $ vagrant ssh frontend -c "sudo su - -c 'yum -y install sshpass'"
            $ vagrant ssh frontend -c "sshpass -p password ssh -o StrictHostKeyChecking=no root@192.168.10.100 '/sbin/ifconfig eth0'"

Access the VM with VNC on the port forwarded by Vagrant from the guest node1:5900 to the host port 15901::

        $ sudo apt-get install -y vinagre
        $ vinagre 127.0.0.1:15901

When done, destroy the test machines with::

        $ vagrant destroy -f


------------
Dependencies
------------

https://github.com/pradels/vagrant-libvirt


-------
License
-------

BSD 2-clause


--------
Problems
--------

1. The following CentOS 7 VM hangs at boot https://forum.opennebula.org/t/centos-7-vm-hangs-at-boot-under-vagrant/1528/1::

        $ vagrant ssh frontend -c "sudo su - oneadmin -c 'oneimage create --name CentOS-7 --path http://marketplace.c12g.com/appliance/53e7bf928fb81d6a69000002/download --driver qcow2 -d default'"
        $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate create --name CentOS-7 --cpu 1 --vcpu 1 --memory 768 --arch x86_64 --disk CentOS-7 --nic private --vnc --ssh ~/.ssh/authorized_keys --net_context'"
        $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate instantiate CentOS-7'"

   Disabling and removing ): apparmor as mentioned at
   https://forum.opennebula.org/t/error-deploying-vm-could-not-create-domain-on-centos-7-kvm/948/11
   did not help, though I swear I saw the VM starting once and logged in through vnc::

        $ sudo service apparmor stop
        $ sudo update-rc.d -f apparmor remove
        $ sudo apt-get -y remove apparmor
        $ sudo reboot

   The one succesfull case may have had something to do with the amount of RAM assigned to the OpenNebula
   worker node and guest, but I'm unable to reproduce this.

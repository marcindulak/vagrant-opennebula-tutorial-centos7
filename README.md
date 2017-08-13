-----------
Description
-----------

Configures OpenNebula cluster using libvirt/kvm nested virtualization in Vagrant.
Vagrant starts a CentOS 7 frontend and OpenNebula worker nodes. The OpenNebula frontend
is then used to manage virtual machines on the OpenNebula worker nodes.

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
                                 e*|
                     192.168.10.100|
                                  ----------
                                  | testvm | ...
                                  ----------


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

and delete the default storage pool::

        $ virsh pool-destroy default

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

- add the started worker nodes (here only node1 is added so we are sure that the testvm virtual machine is started on node1) to the cluster:

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onehost create node1 -i kvm -v kvm'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onehost list'"

- update the system, default image, and files datastores to use `nfs`:

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onedatastore list'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'echo NAME = system > system.one&& echo TM_MAD = shared >> system.one&& echo TYPE = SYSTEM_DS >> system.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onedatastore update system system.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onedatastore list'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'echo NAME = default > default.one&& echo DS_MAD = fs >> default.one&& echo TM_MAD = shared >> default.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onedatastore update default default.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'echo NAME = files > files.one&& echo DS_MAD = fs >> files.one&& echo TM_MAD = shared >> files.one && echo TYPE = FILE_DS >> files.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onedatastore update files files.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onedatastore list'"

- create a network template, consisting of potentially three virtual machines (SIZE = 3):

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'echo NAME = private > private.one; echo VN_MAD = dummy >> private.one; echo BRIDGE = br1 >> private.one; echo AR = [TYPE = IP4, IP = 192.168.10.100, SIZE = 3] >> private.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevnet list'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevnet create private.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevnet list'"

- fetch the CentOS 7 image the from OpenNebula's marketplace::

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'oneimage create --name testvm --path http://marketplace.opennebula.systems/appliance/4e3b2788-d174-4151-b026-94bb0b987cbb/download --datastore default --prefix vd --driver qcow2'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'oneimage list'"

- create a VM template using that image::

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate create --name testvm --cpu 1 --vcpu 1 --memory 256 --arch x86_64 --disk testvm --nic private --vnc --ssh --net_context'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate list'"

- update the template context with the root password (see https://docs.opennebula.org/5.4/operation/vm_setup/kvm.html). Specify `NETWORK=YES` (see https://docs.opennebula.org/5.4/operation/network_management/manage_vnets.html), otherwise opennebula won't configure network on testvm (only lo interface will be present)::

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'echo CONTEXT = [ USERNAME = root, PASSWORD = password, NETWORK = YES  ] > context.one'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate update testvm -a context.one'"

- start the VM using this template::

            $ sleep 30  # wait for template to become ready
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate instantiate testvm'"
            $ sleep 300  # wait for the VM to start
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevm list'"

- collect the information about the host, network, image, template and VM (note that testvm is visible as one-0 to virsh)::

            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onehost show 0'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevnet show 0'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'oneimage show 0'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onetemplate show 0'"
            $ vagrant ssh frontend -c "sudo su - oneadmin -c 'onevm show 0'"
            $ vagrant ssh node1 -c "sudo su - -c 'virsh dumpxml one-0'"

- verify one can ssh to the VM::

            $ vagrant ssh frontend -c "sudo su - -c 'yum -y install sshpass'"
            $ vagrant ssh frontend -c "sshpass -p password ssh -o StrictHostKeyChecking=no root@192.168.10.100 '/sbin/ifconfig'"

Access the first VM with VNC on the port forwarded by Vagrant from the guest node1:5900 to the host port 15900::

        $ sudo apt-get install -y vinagre
        $ vinagre 127.0.0.1:15900

Note that subsequent VMs started on node1 will get 5900 + N as the port number (https://dev.opennebula.org/issues/2045) and won't be accesible in this setup.

The sunstone web-interface is forwared by vagrant from the guest frontend:9869 to the host port 19869, and accessible with credentials (defined in Vagrantfile) `onadmin`/`password`::

        $ firefox 127.0.0.1:19869

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


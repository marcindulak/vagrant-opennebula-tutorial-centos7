# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

ONVER="4.14"

hosts = {
  'frontend' => {'ip' => '192.168.10.5', 'mac' => '52:54:00:00:10:05'},
  'node1' => {'ip' => '192.168.10.11', 'mac' => '52:54:00:00:10:11'},
  'node2' => {'ip' => '192.168.10.12', 'mac' => '52:54:00:00:10:12'}
}

Vagrant.configure(2) do |config|
  config.vm.define 'frontend' do |frontend|
    frontend.vm.box = 'centos/7'
    frontend.vm.box_url = 'centos/7'
    frontend.vm.network 'private_network', ip: hosts['frontend']['ip'], mac: hosts['frontend']['mac'], auto_config: false
    frontend.vm.provider 'libvirt' do |v|
      v.memory = 256
      v.cpus = 1
      v.nested = true
    end
  end
  config.vm.define 'node1' do |node1|
    node1.vm.box = 'centos/7'
    node1.vm.box_url = 'centos/7'
    node1.vm.network 'private_network', ip: hosts['node1']['ip'], mac: hosts['node1']['mac'], auto_config: false
    node1.vm.network 'forwarded_port', guest: 5900, host: 15900
    node1.vm.provider 'libvirt' do |v|
      v.memory = 256
      v.cpus = 1
      v.nested = true
    end
  end
  # disable IPv6 on Linux
  $linux_disable_ipv6 = <<SCRIPT
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1
SCRIPT
  # setenforce 0
  $setenforce_0 = <<SCRIPT
if test `getenforce` = 'Enforcing'; then setenforce 0; fi
#sed -Ei 's/^SELINUX=.*/SELINUX=Permissive/' /etc/selinux/config
SCRIPT
  # stop firewalld
  $systemctl_stop_firewalld = <<SCRIPT
systemctl stop firewalld.service
SCRIPT
  # common settings on all machines
  $etc_hosts = <<SCRIPT
frontend=$1
node1=$2
cat <<END >> /etc/hosts
$frontend frontend
$node1 node1
END
SCRIPT
  $epel7 = <<SCRIPT
yum -y install https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
SCRIPT
  $opennebula_el = <<SCRIPT
ONVER=$1
cat <<END > /etc/yum.repos.d/opennebula.repo
[opennebula]
name=CentOS-$releasever - opennebula
baseurl=http://downloads.opennebula.org/repo/$ONVER/CentOS/\\$releasever/\\$basearch/
enabled=1
gpgcheck=0
END
SCRIPT
  # unattended install_gems
  $root_install_gems_el = <<SCRIPT
cat <<'END' >> /root/install_gems
#!/usr/bin/expect --
spawn /bin/ruby /usr/share/one/install_gems
expect "1. CentOS/RedHat/Scientific"
send "1\r"
expect "Press enter to continue..."
send "\r"
expect "Is this ok \\\[y/d/N\\\]:"
send "y\r"
expect "Press enter to continue..."
send "\r"
set timeout -1
expect "Abort."
puts "Ended expect script."
END
SCRIPT
  # key-based ssh using vagrant keys
  $key_based_ssh = <<SCRIPT
home=`getent passwd $1 | cut -d: -f6`
rm -rf ${home}/.ssh
ls -al ~vagrant ${home}
cp -rp ~vagrant/.ssh ${home}
yum -y install wget
wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O ${home}/.ssh/authorized_keys
SCRIPT
  # .ssh config
  $dotssh_config = <<SCRIPT
home=`getent passwd $1 | cut -d: -f6`
sudo su -c "cat << EOF > ${home}/.ssh/config
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
EOF"
sudo su -c "chmod 600 ${home}/.ssh/config"
SCRIPT
  # .ssh permission
  $dotssh_chmod_600 = <<SCRIPT
home=`getent passwd $1 | cut -d: -f6`
sudo su -c "chmod -R 600 ${home}/.ssh"
sudo su -c "chmod 700 ${home}/.ssh"
SCRIPT
  # .ssh ownership
  $dotssh_chown = <<SCRIPT
home=`getent passwd $1 | cut -d: -f6`
sudo su -c "chown -R $1:$2 ${home}/.ssh"
SCRIPT
  # configure the bridge
  $ifcfg_bridge = <<SCRIPT
DEVICE=$1
BRIDGE=$2
HWADDR=$3
cat <<END >> /etc/sysconfig/network-scripts/ifcfg-$DEVICE
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
DEVICE=$DEVICE
HWADDR=$HWADDR
PEERDNS=no
TYPE=Ethernet
BRIDGE=$BRIDGE
END
ARPCHECK=no /sbin/ifup $DEVICE 2> /dev/null
SCRIPT
  # configure the second vagrant eth interface
  $ifcfg = <<SCRIPT
IPADDR=$1
NETMASK=$2
DEVICE=$3
TYPE=$4
cat <<END >> /etc/sysconfig/network-scripts/ifcfg-$DEVICE
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
IPADDR=$IPADDR
NETMASK=$NETMASK
DEVICE=$DEVICE
PEERDNS=no
TYPE=$TYPE
END
ARPCHECK=no /sbin/ifup $DEVICE 2> /dev/null
SCRIPT
  config.vm.define "frontend" do |frontend|
    # check for nested virtualization!
    frontend.vm.provision :shell, :inline => 'egrep -c "(vmx|svm)" /proc/cpuinfo'
    frontend.vm.provision :shell, :inline => 'hostname frontend', run: 'always'
    frontend.vm.provision 'shell' do |s|
      s.inline = $etc_hosts
      s.args   = hosts['frontend']['ip'] + ' ' + hosts['node1']['ip']
    end
    frontend.vm.provision :shell, :inline => $linux_disable_ipv6
    frontend.vm.provision :shell, :inline => $setenforce_0
    frontend.vm.provision :shell, :inline => $epel7
    frontend.vm.provision 'shell' do |s|
      s.inline = $opennebula_el
      s.args   = ONVER
    end
    frontend.vm.provision :shell, :inline => 'yum -y install opennebula-server opennebula-sunstone'
    frontend.vm.provision :shell, :inline => $root_install_gems_el
    frontend.vm.provision :shell, :inline => 'yum -y install expect'
    frontend.vm.provision :shell, :inline => 'expect /root/install_gems'
    # configure key-based ssh for oneadmin user using vagrant's keys
    frontend.vm.provision :file, source: '~/.vagrant.d/insecure_private_key', destination: '~vagrant/.ssh/id_rsa'
    frontend.vm.provision 'shell' do |s|
      s.inline = $key_based_ssh
      s.args   = 'oneadmin'
    end
    frontend.vm.provision 'shell' do |s|
      s.inline = $dotssh_chmod_600
      s.args   = 'oneadmin'
    end
    frontend.vm.provision 'shell' do |s|
      s.inline = $dotssh_config
      s.args   = 'oneadmin'
    end
    frontend.vm.provision 'shell' do |s|
      s.inline = $dotssh_chown
      s.args   = 'oneadmin oneadmin'
    end
    frontend.vm.provision 'shell' do |s|
      s.inline = $ifcfg_bridge
      s.args   = 'eth1 br1 ' + hosts['frontend']['mac']
    end
    frontend.vm.provision :shell, :inline => 'yum -y install bridge-utils'
    frontend.vm.provision 'shell' do |s|
      s.inline = $ifcfg
      s.args   = hosts['frontend']['ip'] + ' 255.255.255.0 br1 Bridge'
    end
    frontend.vm.provision :shell, :inline => 'ifup eth1', run: 'always'
    frontend.vm.provision :shell, :inline => 'ifup br1', run: 'always'
    frontend.vm.provision :shell, :inline => 'systemctl start opennebula'
    frontend.vm.provision :shell, :inline => 'systemctl enable opennebula'
    frontend.vm.provision :shell, :inline => 'systemctl start opennebula-sunstone'
    frontend.vm.provision :shell, :inline => 'systemctl enable opennebula-sunstone'
    frontend.vm.provision :shell, :inline => 'echo "/var/lib/one/ *(rw,sync,no_subtree_check,no_root_squash)" > /etc/exports'
    frontend.vm.provision :shell, :inline => 'systemctl start rpcbind nfs-server'
    frontend.vm.provision :shell, :inline => 'systemctl enable rpcbind nfs-server'
    frontend.vm.provision :shell, :inline => 'sudo su - oneadmin -c "sleep 10&& onevm list"'
  end
  config.vm.define 'node1' do |node1|
    # check for nested virtualization!
    node1.vm.provision :shell, :inline => 'egrep -c "(vmx|svm)" /proc/cpuinfo'
    node1.vm.provision :shell, :inline => 'hostname node1', run: 'always'
    node1.vm.provision 'shell' do |s|
      s.inline = $etc_hosts
      s.args   = hosts['frontend']['ip'] + ' ' + hosts['node1']['ip']
    end
    node1.vm.provision :shell, :inline => $linux_disable_ipv6
    node1.vm.provision :shell, :inline => $setenforce_0
    node1.vm.provision :shell, :inline => $epel7
    node1.vm.provision 'shell' do |s|
      s.inline = $opennebula_el
      s.args   = ONVER
    end
    node1.vm.provision :shell, :inline => 'yum -y install opennebula-node-kvm'
    node1.vm.provision 'shell' do |s|
      s.inline = $ifcfg_bridge
      s.args   = 'eth1 br1 ' + hosts['node1']['mac']
    end
    node1.vm.provision :shell, :inline => 'yum -y install bridge-utils'
    node1.vm.provision 'shell' do |s|
      s.inline = $ifcfg
      s.args   = hosts['node1']['ip'] + ' 255.255.255.0 br1 Bridge'
    end
    #end
    node1.vm.provision :shell, :inline => 'ifup eth1', run: 'always'
    node1.vm.provision :shell, :inline => 'ifup br1', run: 'always'
    node1.vm.provision :shell, :inline => 'echo frontend:/var/lib/one/  /var/lib/one/  nfs   soft,intr,rsize=8192,wsize=8192,noauto >> /etc/fstab'
    node1.vm.provision :shell, :inline => 'mount /var/lib/one/'
    node1.vm.provision :shell, :inline => 'systemctl start libvirtd'
    node1.vm.provision :shell, :inline => 'systemctl enable libvirtd'
  end
end

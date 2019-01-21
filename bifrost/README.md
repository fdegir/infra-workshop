# Table of Contents

1. [Introduction](#introduction)
2. [Prepare Host Machine](#prepare-host)  
3. [Creating Virtual Machineis](#create-vms)  
4. [Bifrost Installation](#bifrost-installation)  


# Introduction <a name="introduction"></a>

This guide contains information about provisioning nodes using **Bifrost**
and tailored for the objectives of the workshop and the environment it is
expected to be run on.

Information in this guide is based on official Bifrost documentation and
adapted for the purposes of this workshop. [1] Information from OPNFV XCI
is also incorporated into the documentation. [2]

All the commands listed on this guide must be executed on Jumphost unless
otherwise is noted.

Please note that certain parts in configuration files or the playbooks
are left empty and they are expected to be filled by the participants.

# Prepare Jumphost <a name="prepare-host"></a>

This section contains the installation and configuration steps that must be
performed on jumphost. Please copy and paste the commands to your terminal
where you are logged in to jumphost.

Before we start with the rest of the workshop, you can clone the workshop
git repo from Github in order to use various files that have been created
in advance in order not to spend time creating or modifying them manually.

```bash
git clone https://github.com/fdegir/infra-workshop.git $HOME/infra-workshop
```

As usual, we update the system, install required packages and fix some
of the issues in advance before we face them.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y gcc libffi-dev libssl-dev lsb-release make net-tools \
  python-pip python-minimal libpython-dev python-yaml wget curl apt-utils \
  python-selinux libvirt-bin qemu-utils qemu-kvm qemu-system-x86 virt-what virtinst \
  sgabios libvirt-dev
sudo apt purge -y python-openssl # python-openssl causes issues so purge it
sudo hostnamectl set-hostname bifrost
sudo bash -c "sed -i -e '/127.0.0.1/s/$/ bifrost/' /etc/hosts"
```

Our SSH key will be injected to the built image in order for us to login
to them so we need to generate keys.

```bash
ssh-keygen -t rsa -q -P '' -f ${HOME}/.ssh/id_rsa
```

We can reboot the machine and continue with the next section.

```bash
sudo reboot
```

# Creating Virtual Machines <a name="create-vms"></a>

The workshop will use virtual machines for provisioning and bootstrapping.
The operating system the VMs will be provisioned with is Ubuntu 16.04 and
it is important these machines have sufficient performance to achieve this.

Nested virtualization will help in this case since the machine each participant
is provided with is created on an OpenStack cloud and the machines we will
be provisioning and bootstrapping are running within that instance. (L2 guest) [3]

In order for us to be able to use nested virtualization, we must ensure
it is supported by the CPU meaning that CPU virtualization extensions are
available and nested virtualization is enabled. [4]

```bash
grep -i -E "vmx|svm" /proc/cpuinfo
cat /sys/module/kvm_intel/parameters/nested
lsmod | grep kvm
```

As you will see in the output of the commands above, virtualization
extensions on the machine are available and nested virtualization is
enabled so we can continue with the next steps, first checking to see
if libvirtd service is running and we are member of libvirtd group
so we can interact with it.

```bash
sudo systemctl status libvirtd
id -a | grep libvirtd
```

We need to enable ip forwarding for the libvirt bridge to operate properly
with dnsmasq.

```bash
sudo sysctl -p
sudo bash -c "sed -i '/^#net.ipv4.ip_forward/s/^#//g' /etc/sysctl.conf"
sudo systemctl daemon-reload
sudo sysctl -p
```

Ubuntu packaging+apparmor issue prevents libvirt from loading the ROM
from /usr/share/misc so we need to ensure sgabios.bin is available in
where it can be loaded.

```bash
ls -al /usr/share/qemu/sgabios.bin # if the file doesn't exist, sudo cp /usr/share/misc/sgabios.bin /usr/share/qemu/sgabios.bin
```

The default network created by libvirt during its installation conflicts
with what we will be doing so we need to remove that network and create
one for us to use for PXE booting.

```bash
virsh net-list --all
ip a s
virsh net-destroy default
virsh net-undefine default
virsh net-define $HOME/infra-workshop/bifrost/files/pxe-network.xml
virsh net-autostart pxe
virsh net-start pxe
virsh net-list --all
ip a s # you should see br-pxe and not virbr0
```

We now need to create a new pool in order for us to create volumes
to use for our VMs. and volumes themselves.

```bash
virsh pool-define $HOME/infra-workshop/bifrost/files/images-pool.xml
virsh pool-autostart images
virsh pool-start images
virsh pool-list --all
```

And volumes themselves.

```bash
virsh vol-create-as images controller00.qcow2 10G --format qcow2 --prealloc-metadata
virsh vol-create-as images compute00.qcow2 10G --format qcow2 --prealloc-metadata
sudo chattr +C /var/lib/libvirt/images/controller00.qcow2 # set copy-on-write for volume on non-CentOS systems
sudo chattr +C /var/lib/libvirt/images/compute00.qcow2 # set copy-on-write for volume on non-CentOS systems
virsh vol-list --pool images
```

VM console logs could be useful when we need to troubleshoot stuff so
we create a folder to store the logs.

```bash
mkdir /tmp/logs
```

We have everything ready for us to create our VMs.

VMs can be created in different ways, using virt-install or virsh
from an existing XML file. For the sake of time, we will use precreated
XML files to create VMs. Please take a look at them to see how
they look. The 2 important parts in VM configuration are the boot
order

```xml
  <os>
    <type arch='x86_64' machine='pc-i440fx-2.5'>hvm</type>
    <boot dev='network'/>
    <bootmenu enable='no'/>
    <bios useserial='yes' rebootTimeout='10000'/>
  </os>
```

and the network the interface is attached to.

```xml
    <interface type='network'>
      <source network='pxe' bridge='br-pxe'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
```

Now, issue below commands to define the VMs. Please do not start
them manually as it will be done by bifrost itself via power control
with the help of Virtual BMC.

```bash
virsh define $HOME/infra-workshop/bifrost/files/controller00.xml
virsh define $HOME/infra-workshop/bifrost/files/compute00.xml
virsh list --all
```

In order for the workshop to be realistic, the machine power control
will be done over IPMI instead of virsh even though we are working
with VMs using Virtual BMC (vbmc). [5] [6]

vbmc is a utility created by OpenStack community to control VMs
using IPMI commands, simulating Baseboard Management Controller (BMC).

```bash
sudo pip install virtualbmc==1.3
sudo vbmc add controller00 --libvirt-uri qemu:///system --port 625
sudo vbmc add compute00 --libvirt-uri qemu:///system --port 626
sudo vbmc start controller00
sudo vbmc start compute00
sudo vbmc list
```

And finally, we need to create inventory for bifrost so it knows
which machines it is operating on.

```bash
cp $HOME/infra-workshop/bifrost/files/baremetal.json /tmp
```

**Do not forget to add vbmc ports and MAC addresses of your VMs
into this file. Or bifrost will fail provisioning machines!**

You can find out vbmc ports and mac addresses of your VMs using
the commands below.

```bash
sudo vbmc list
virsh dumpxml controller00 | grep 'mac address'
virsh dumpxml compute00 | grep 'mac address'
```

# Bifrost Installation <a name="bifrost-installation"></a>

Bifrost installation is pretty straightforward since everything
is automated. Apart from being automated, it allows users to
configure it in a way that suits their needs best.

First, we need to clone bifrost repository and checkout a known
working version.

Next step is to install Ansible and its dependencies since Bifrost is Ansible
based. Once the installation is done, we can reboot our machine.

```bash
sudo pip install -U pip==9.0.3
sudo pip install ansible==2.5.8 virtualbmc
```

```bash
cd $HOME
git clone https://git.openstack.org/openstack/bifrost
cd bifrost && git checkout 0f605cd723
```

As noted above, we need to set few environment variables to
configure how bifrost should work. Some of these environment
variables are necessary for Disk Image Builder bifrost uses
for building images.

```bash
export DIB_OS_RELEASE="xenial"
export DIB_OS_ELEMENT="ubuntu-minimal"
export DIB_OS_PACKAGES="vim,less,bridge-utils,language-pack-en,iputils-ping,rsyslog,curl,iptables"
export BIFROST_INVENTORY_SOURCE=/tmp/baremetal.json
export BIFROST_INVENTORY_DHCP=false
export BIFROST_DOWNLOAD_IPA=true
export BIFROST_CREATE_IPA=false
```

sudo mkdir /httpboot /tftpboot
sudo mkdir /etc/dnsmasq.d/bifrost.dhcp-hosts.d
sudo chmod -R 0755 /etc/dnsmasq.d/bifrost.dhcp-hosts.d
cd $HOME/bifrost
sudo pip install --upgrade -r requirements.txt
ansible-playbook -i inventory/target ws-install.yaml

ansible-playbook -i inventory/bifrost_inventory.py enroll-dynamic.yaml
ansible-playbook -i inventory/bifrost_inventory.py deploy-dynamic.yaml

# Next Steps <a name="next-steps"></a>

# References <a name="references"></a>

1. https://docs.openstack.org/bifrost/latest/
2. https://opnfv-releng-xci.readthedocs.io/en/latest/
3. https://www.linux-kvm.org/page/Nested_Guests
4. https://en.wikipedia.org/wiki/X86_virtualization
5. https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface
6. https://docs.openstack.org/virtualbmc/latest/

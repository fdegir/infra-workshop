# Table of Contents

1. [Introduction](#introduction)
2. [Prepare Host Machine](#prepare-host)  
3. [Creating Virtual Machines](#create-vms)  
4. [Bifrost Installation](#bifrost-installation)  
5. [Provisioning Machines](#provisioning-machines)  
6. [Bootstrapping Machines](#bootstrapping-machines)  
7. [Cleanup](#cleanup)  


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
with what we will be doing so we need to remove that network.

```bash
virsh net-list --all
ip a s
virsh net-destroy default
virsh net-undefine default
```

The VMs we will create and provision will have 3 network interfaces
connected to pxe/admin, management, and neutron external networks so
we create those networks.

```bash
networks="pxe mgmt ext"
for network in $networks; do
virsh net-define $HOME/infra-workshop/bifrost/files/${network}-network.xml
virsh net-autostart $network
virsh net-start $network
done
virsh net-list --all
ip a s # you should see br_pxe, br_mgmt, br_ext and not virbr0
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

and the interface that is attached to pxe network.

```xml
    <interface type='network'>
      <source network='pxe' bridge='br_pxe'/>
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
virsh dominfo controller00
virsh dominfo compute00
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
virsh domiflist controller00
virsh domiflist compute00
```

# Bifrost Installation <a name="bifrost-installation"></a>

Bifrost installation is pretty straightforward since everything
is automated. Apart from being automated, it allows users to
configure it in a way that suits their needs best.

First, we need to clone bifrost repository,  checkout a known
working version, install its requirements, and finally install
Ansible.


```bash
git clone https://git.openstack.org/openstack/bifrost $HOME/bifrost
cd bifrost && git checkout 0f605cd723
sudo pip install --upgrade -r requirements.txt
sudo pip install ansible==2.5.8
```

Bifrost configuration can either directly be done within corresponding
vars files, from command line or using environment variables. We will
set few environment variables to achieve this.

Bifrost uses Diskimage-builder (DIB) for building customized operating
system images so first few variables are used for telling Bifrost and
indirectly DIB about what kind of image we want to build. [7]

```bash
export DIB_OS_RELEASE="xenial"
export DIB_OS_ELEMENT="ubuntu-minimal"
export DIB_OS_PACKAGES="vim,less,bridge-utils,language-pack-en,iputils-ping,rsyslog,curl,iptables"
```

Next environment variables are used for Bifrost itself.

```bash
export BIFROST_INVENTORY_SOURCE=/tmp/baremetal.json
export BIFROST_INVENTORY_DHCP=false
export BIFROST_DOWNLOAD_IPA=true
export BIFROST_CREATE_IPA=false
```

Please note that if you logout from your machine or close the console
you will need to set these environment variables again before interacting
with bifrost again.

Bifrost operates dnsmasq itself so existing dnsmasq processes cause issues
for it. Before we proceed with Bifrost installation, we need to kill all
dnsmasq processes.

```bash
sudo killall -w dnsmasq
pgrep dnsmasq
```

Bifrost installation can be done with existing playbooks but one can create
their own playbooks to do customized installations using Bifrost roles. This
is what we will be doing so please diff original role with ours to see how
we configure our installation.

```bash
cp $HOME/infra-workshop/bifrost/files/ws-install.yaml $HOME/bifrost/playbooks/
diff $HOME/bifrost/playbooks/install.yaml cp $HOME/infra-workshop/bifrost/playbooks/ws-install.yaml
cd $HOME/bifrost/playbooks
ansible-playbook -i inventory/target ws-install.yaml
```

The installation and build of the operating system image will take about 20
minutes so this is a good opportunity to take a coffee break. We will continue
with provisioning the machines when we are back.

# Provisioning Machines <a name="provisioning-machines"></a>

At this point, you can open 2 additional console windows to monitor the
provisioning.

On first console window, login to your machine and issue below command
to see your nodes being enrolled, managed by Bifrost via IPMIs (powered
up, etc).

```bash
source $HOME/bifrost/env-vars
for i in {1..500}; do
clear
ironic node-list
echo
virsh list --all
sleep 2
done
```

On second console window, login to your machine and issue below command
to see the traffic going through the bridge, br_pxe, which will show
all the DHCP, and boot requests coming from VMs and so on.

```bash
sudo tcpdump -i br_pxe
```

The first thing we need to do is to enroll our nodes in Ironic using Bifrost.
Once the nodes are enrolled, it will be in **available** state for further
operation with Ironic.

```bash
cd $HOME/bifrost/playbooks
ansible-playbook -vvv -i inventory/bifrost_inventory.py enroll-dynamic.yaml -e network_interface=br_pxe
```

You should see your nodes to appear in the output of ironic command on
the first console.

We can start deployment now.

```bash
cd $HOME/bifrost/playbooks
ansible-playbook -vvv -i inventory/bifrost_inventory.py deploy-dynamic.yaml -e network_interface=br_pxe
```

On first console, you should see the different states nodes go through, change
in their power states and so on. You can see the different states on ironic
documentation. [8]

Once the nodes are powered on, you should start seeing traffic
on your console where you are running tcpdump. Please look at
the output and get yourself familiar with it since it will be useful
when we do this for our lab.

Wait until you see your nodes in active state in ironic command output which
will take about 5 minutes.

Once you see your nodes as **active** on ironic output, you can ssh to them.

```bash
ssh root@<ip_of_controller00>
ssh root@<ip_of_compute00>
```

# Bootstrapping Machines <a name="bootstrapping-machines"></a>

In this section, we will do basic configuration on the nodes we just
provisioned.

* adding routes so nodes can access to the internet
* applying network configuration
* configuring time sync
* updating the system
* installing packages
* rebooting the nodes and waiting for them to come back online

But before starting with the above tasks, we can test access to the
nodes via Ansible.

```bash
export BIFROST_INVENTORY_SOURCE=/tmp/baremetal.json
cat $BIFROST_INVENTORY_SOURCE
ansible -u root -i $HOME/bifrost/playbooks/inventory/bifrost_inventory.py baremetal -a "uname -an"
```

There are 3 things to note above which we need to keep in mind while
going through the rest of the workshop.

* **user**: As you might have noticed above, we are instructing Ansible
to operate under **root** user. This is needed since our SSH public
key is baked into the operating system image for root user.
* **inventory**: We already have the inventory created for Bifrost
to provision the nodes. This inventory already contains the required
information regarding the nodes so we do not need to create a separate
inventory and just continue using it with dynamic inventory mechanism
made available by Bifrost. [9] Please make sure that the environment
variable **BIFROST_INVENTORY_SOURCE** is always set, pointing to the
right bifrost inventory file before you issue ansible commands.
* **host groups**: We will be operating on the nodes we provisioned
so the tasks we are about to execute must be performed on those and
not somewhere else. We can specify them by using **baremetal** group
which our target nodes controller00 and compute00 are members of.
You can check this by looking into baremetal.json file.

If the last ansible command worked fine, you are now ready to
bootstrap nodes by issuing the command below.

```bash
ansible-playbook -i $HOME/bifrost/playbooks/inventory/bifrost_inventory.py $HOME/infra-workshop/bifrost/playbooks/configure-targethosts.yml
```

This command will take few minutes to complete and you will see what
Ansible is doing on the nodes on your console.

The hostnames Ansible operates on are displayed after each task since
we instructed Ansible to operate on hosts within baremetal group and
the hosts in this group are controller00 and compute00, bootstrapped
by Ansible.

As you will notice, a specific role is used for bootstrapping nodes.
Roles provide a framework for fully independent, or interdependent
collections of variables, tasks, files, templates, and modules. Roles
also can enable the reuse so one role can be used within different
playbooks. [10]

# Cleanup

We are nearly done with the workshop but before we end it, we could
perhaps cleanup the environment by

```bash
source $HOME/bifrost/env-vars
ironic node-list
ironic node-set-provision-state 00000000-0000-0000-0000-000000000001 deleted
ironic node-set-provision-state 00000000-0000-0000-0000-000000000002 deleted
ironic node-delete 00000000-0000-0000-0000-000000000001 # issue this command when the node becomes available
ironic node-delete 00000000-0000-0000-0000-000000000002 # issue this command when the node becomes available
ironic node-list
sudo vbmc list
sudo vbmc delete controller00
sudo vbmc delete compute00
sudo vbmc list
virsh list --all
virsh destroy controller00
virsh destroy compute00
virsh undefine controller00
virsh undefine compute00
virsh list --all
```

# References <a name="references"></a>

1. https://docs.openstack.org/bifrost/latest/
2. https://opnfv-releng-xci.readthedocs.io/en/latest/
3. https://www.linux-kvm.org/page/Nested_Guests
4. https://en.wikipedia.org/wiki/X86_virtualization
5. https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface
6. https://docs.openstack.org/virtualbmc/latest/
7. https://docs.openstack.org/diskimage-builder/latest/
8. https://docs.openstack.org/ironic/latest/contributor/states.html
9. https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html
10. https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html

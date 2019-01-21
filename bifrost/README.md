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
adapted for the purposes of this workshop. [1]

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

Before doing anything else, it is a good idea to update the system, install
required packages and fix some annoying warning messages.

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
to it so we need to generate keys.

```bash
ssh-keygen -t rsa -q -P '' -f ${HOME}/.ssh/id_rsa
```

We can reboot the machine and continue with the next section.

```bash
sudo reboot
```

# Creating Virtual Machines <a name="create-vms"></a>

In this section, we will create the virtual machines we will provision
and bootstrap for the rest of the workshop.
The workshop will use KVM so it is necessary to ensure CPU virtualization
extensions are available and nested virtualization is enabled.

```bash
grep -i -E "vmx|svm" /proc/cpuinfo
cat /sys/module/kvm_intel/parameters/nested
lsmod | grep kvm
```

As you will see in the output of the commands above, virtualization
extensions on the machine are available but nested virtualization is
not enabled. Please issue the commands below to enable nested virtualization.

```bash
sudo modprobe -r kvm_intel
sudo modprobe kvm_intel nested=1
cat /sys/module/kvm_intel/parameters/nested
```

The command above doesn't make the setting permanent so if the machine
gets rebooted, the nested virtualization will be disabled. In order
to make the change permanent, please use the command below to update
the file **/etc/modprobe.d/qemu-system-x86.conf**.

```bash
sudo bash -c 'cat << EOF > /etc/modprobe.d/qemu-system-x86.conf
options kvm-intel nested=y
options kvm-intel enable_apicv=n
EOF'
```

Since the workshop will use VMs created using libvirt, we need to
install required packages, and enable & start libvirtd service. Please
answer N to the question you'll be asked during package installation.

```bash
TODO: ssh-keygen -t rsa?????
TODO: chmod -R go-rwx .ssh
TODO: sudo cp -R $HOME/.ssh /root/.ssh
TODO: libguestfs-tools???????

sudo systemctl start libvirtd
sudo systemctl enable libvirtd
sudo systemctl status libvirtd
sudo usermod -aG libvirtd ubuntu
```

Please logout and log back in in order for group change to take effect.

We can now check the available VMs, networks, and storage pools.

```bash
virsh list --all
virsh net-list --all
virsh pool-list --all
```


We will create the VMs in 2 different ways; using virt-install and
virsh.

Before creating the VMs, we need to create disks for them.

```bash
mkdir /home/ubuntu/images
qemu-img create -f qcow2 /home/ubuntu/images/controller00.qcow2 10G
qemu-img create -f qcow2 /home/ubuntu/images/compute00.qcow2 10G
```

Let's start with virt-install. Below command creates VM controller00
with the details specified in the command. As you will see there, the
boot device is set as PXE.

```bash
virt-install --name controller00 \
  --virt-type kvm \
  --vcpus 1 \
  --cpu host-model \
  --memory 1024 \
  --os-type linux \
  --disk /home/ubuntu/images/controller00.qcow2,format=qcow2,size=10,bus=virtio \
  --network network=default,model=virtio \
  --boot network,hd,menu=off,useserial=yes \
  --events on_poweroff=destroy,on_reboot=restart,on_crash=restart \
  --graphics none \
  --hvm \
  --noautoconsole
```

After creating the VM controller00, we need to edit the domain and ensure the VM
reboots until it finds a boot device. Please issue the command below

```bash
virsh destory controller00
virsh edit controller00
```

and update the line

```bash
<bios useserial='yes' />
```

as

```bash
<bios useserial='yes' rebootTimeout='10000' />
```

save and start the domain.

```bash
virsh start controller00
```

Second VM, compute00, will be created using virsh.

```bash
virsh define $HOME/infra-workshop/bifrost/xml/compute00.xml
virsh start compute00
```

Let us look at the VMs we created.

```bash
virsh list --all
```

We can also see that the VMs attempt to do PXE-boot, fails and
they get restarted due to the setting **rebootTimeout** we set.

```bash
sudo tcpdump -i virbr0
```

We not have our VMs ready to be provisioned and bootstrapped so
we can proceed with Bifrost installation and configuration.

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

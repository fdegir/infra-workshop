# Table of Contents

1. [Introduction](#introduction)
2. [Prepare Jumphost](#prepare-jumphost)  


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

# Prepare Jumphost <a name="prepare-jumphost"></a>

This section contains the installation and configuration steps that must be
performed on jumphost. Please copy and paste the commands to your terminal
where you are logged in to jumphost.

Before we start with the rest of the workshop, you can clone the workshop
git repo from Github in order to use various files that have been created
in advance in order not to spend time creating or modifying them manually.

```bash
git clone https://github.com/fdegir/infra-workshop.git
```

Before doing anything else, it is a good idea to update the system

```bash
sudo apt update && sudo apt upgrade -y
```

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
sudo apt install -y libvirt-bin
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
sudo systemctl status libvirtd
sudo usermod -aG libvirt ubuntu
```

Please logout and log back in in order for group change to take effect.

We can now check the available VMs, networks, and storage pools.

```bash
virsh list --all
virsh net-list --all
virsh pool-list --all
```

Next step is to install Ansible and its dependencies since Bifrost is Ansible
based. Once the installation is done, we can reboot our machine.

```bash
sudo apt-get install -y python-pip python-dev libffi-dev gcc \
    libssl-dev python-selinux
sudo pip install -U pip==9.0.3
sudo pip install ansible==2.5.8
```

# Next Steps <a name="next-steps"></a>

# References <a name="references"></a>

1. https://docs.openstack.org/bifrost/latest/

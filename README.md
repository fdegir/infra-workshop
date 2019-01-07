# Table of Contents

1. [Introduction](#introduction)
2. [Prepare Jumphost](#prepare-jumphost)
3. [Next Steps](#next-steps)

# Introduction <a name="introduction"></a>

# Prepare Jumphost <a name="prepare-jumphost"></a>

This section contains the installation and configuration steps that must be
performed on jumphost. The steps documented here are common for Kolla-Ansible
and Kubespray. Please copy and paste the commands to your terminal where you
are logged in to jumphost.

Before we start with the rest of the workshop, you can clone the workshop
git repo from Github in order to use various files that have been created
in advance in order not to spend time creating or modifying them manually.

```bash
git clone https://github.com/fdegir/infra-workshop.git
```

## Install Ansible <a name="install-ansible"></a>

Both [Kolla Ansible](https://docs.openstack.org/kolla-ansible/queens/index.html)
and [Kubespray](https://github.com/kubernetes-incubator/kubespray) use
[Ansible](https://www.ansible.com/) for installation and configuration of
OpenStack and Kubernetes respectively.

This section contains instructions to install and configure Ansible version
**2.5.8** on jumphost.

We first install the packages required by Ansible.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt-get install -y python-pip python-dev libffi-dev gcc \
    libssl-dev python-selinux
sudo pip install -U pip==9.0.3
```

We can now install Ansible.

```bash
sudo pip install ansible==2.5.8 # Kubespray has issues with 2.7 so we choose a known version
```

We can use existing inventory to install python-dev package on target nodes
and test Ansible installation.

```bash
cat $HOME/infra-workshop/sample-inventory
ansible -i $HOME/infra-workshop/sample-inventory all -m raw -a "apt-get install -y python-dev"
ansible -i $HOME/infra-workshop/sample-inventory all -a "uname -an"
```

# Next Steps <a name="next-steps"></a>

If everything went fine until this point, it means you are now ready to
move to **OpenStack** part of the workshop. Please open the instructions by
clicking [this link](https://github.com/fdegir/infra-workshop/tree/master/openstack).

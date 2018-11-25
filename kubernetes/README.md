# Table of Contents

1. [Introduction](#introduction)
2. [Kubernetes Installation](#kubernetes-installation)  
  2.1. [Clone Kubespray Repo and Configure Kubespray](#clone-configure-kubespray)  
  2.2. [Install Kubernetes](#install-kubernetes)  
  2.3. [Prepare for Initial Use](#prepare-initial-use)  
3. [Basic Kubernetes Usage](#use-kubernetes)  
  3.1. [Use Kubernetes with kubectl](#use-kubernetes-with-kubectl)  
  3.2. [Use Kubernetes with Kubernetes Dashboard](#use-kubernetes-with-dashboard)  
4. [Next Steps](#next-steps)
5. [References](#references)


# Introduction <a name="introduction"></a>

This guide contains information about installing **Kubernetes 1.11.3** using
Kubespray. Almost everything in this guide is taken from official Kubespray
documentation and adapted for the purposes of this workshop. [1]

All the commands listed on this guide must be executed on Jumphost unless
otherwise is noted.

# Kubernetes Installation <a name="kubernetes-installation"></a>

## Clone Kubespray Repo and Configure Kubespray <a name="clone-configure-kubespray"></a>

Kubespray contains Ansible playbooks and roles to install Kubernetes. In order
for us to be able to use them, we first need to clone its repo.

Another important point is that we will use a released version of Kubespray
so we need to check out a certain tag rather than using the tip of the master.

```bash
cd $HOME/infra-workshop/kubernetes
git clone -b v2.7.0 https://github.com/kubernetes-incubator/kubespray.git
cd kubespray
git show
```

In order for Kubespray to function correctly, we need to install its
dependencies.

```bash
sudo pip install -r requirements.txt
```

Kubespray requires inventory to operate on nodes so we need to generate
inventory file for our environment. In order to save some time, an inventory
file named hosts.ini is available in kubernetes directory which we can use
during Kubernetes installation with Kubespray. It would be useful if you take
a look at it to see how it looks.

```bash
cp -rfp inventory/sample inventory/ws
cp -f ../hosts.ini inventory/ws/hosts.ini
cat inventory/ws/multinode
```

Kubespray installation configuration is kept in **inventory/ws/inventory/ws/group_vars/k8s-cluster/k8s-cluster.yml**.
We will use a configuration file created for the purposes of this workshop
named k8s-cluster.yml and located in kubernetes directory.

```bash
cp -f ../k8s-cluster.yml inventory/ws/group_vars/k8s-cluster/k8s-cluster.yml
cat inventory/ws/group_vars/k8s-cluster/k8s-cluster.yml
```

## Install Kubernetes <a name="install-kubernetes"></a>

We are now ready to start installing Kubernetes using Kubespray.

```bash
ansible-playbook -i inventory/ws/hosts.ini --become --become-user=root cluster.yml
```

## Prepare for Initial Use <a name="prepare-initial-use"></a>

TBD

# Basic Kubernetes Usage <a name="use-kubernetes"></a>

## Use Kubernetes with kubectl <a name="use-kubernetes-with-kubectl"></a>

TBD

## Use Kubernetes with Kubernetes Dashboard <a name="use-kubernetes-with-dashboard"></a>

TBD

# Next Steps <a name="next-steps"></a>

TBD

# References <a name="references"></a>

1. https://github.com/kubernetes-incubator/kubespray

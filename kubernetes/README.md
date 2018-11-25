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
cat inventory/ws/hosts.ini
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

The installation could take up to 20 minutes.

```bash
ansible-playbook -i inventory/ws/hosts.ini --become --become-user=root cluster.yml
```

## Prepare for Initial Use <a name="prepare-initial-use"></a>

Kubespray can be configured to install kubectl but due to lack of memory on
jumphost, we will install it using package manager.

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

Kubectl expects Kubernetes cluster details to be available as **~/.kube/config** so
we need to make this directory and copy it from kubespray folder to there.

```bash
kubectl get nodes # <-- this will fail due to missing cluster details
mkdir $HOME/.kube
cp inventory/ws/artifacts/admin.conf ~/.kube/config
```

# Basic Kubernetes Usage <a name="use-kubernetes"></a>

## Use Kubernetes with kubectl <a name="use-kubernetes-with-kubectl"></a>

In this part of the workshop we will work with below Kubernetes resource types
using kubectl. [2]

* deployments
* services
* pods
* namespaces

Kubectl accepts commands in below form

```bash
kubectl [command] [TYPE] [NAME] [flags]
```

Before doing anything else, we can start by listing available Kubernetes
clusters, nodes, contexts, namespaces, and pods.

```bash
kubectl config get-clusters
kubectl get nodes
kubectl config get-contexts
kubectl get namespaces
```

We can also check if anything is available for use in our Kubernetes instance.

```bash
kubectl get deployment --all-namespaces
kubectl get service --all-namespaces
kubectl get pod --all-namespaces
```

As you will notice, Kubernetes services are deployed on Kubernetes itself.

We will now deploy a microservices demo application on our Kubernetes cluster.
This application is named [Sock Shop](https://microservices-demo.github.io/)
and available on Github.

```bash
kubectl create namespace sock-shop
cd $HOME/infra-workshop/kubernetes
git clone https://github.com/microservices-demo/microservices-demo.git
cd microservices-demo/deploy/kubernetes/
kubectl apply -f complete-demo.yaml
```

It could take 2 minutes for application to be deployed.

You can follow the progress by using the command below

```bash
for i in {1..100}; do
clear
kubectl get pod -n sock-shop
sleep 2
done
```

Once all the PODs are in running state, you can kill the running command
by Ctrl+C.

Let's see what has been created.

```bash
kubectl get pod -n sock-shop
kubectl get deployment -n sock-shop
kubectl get service -n sock-shop
```

We can get more details about the resource types created.

```bash
kubectl describe pod user-7848fb86db-jqlgh -n sock-shop # <-- pick one of your PODs
kubectl describe deployment orders -n sock-shop
kubectl describe service front-end -n sock-shop
```








## Use Kubernetes with Kubernetes Dashboard <a name="use-kubernetes-with-dashboard"></a>

TBD

# Next Steps <a name="next-steps"></a>

TBD

# References <a name="references"></a>

1. https://github.com/kubernetes-incubator/kubespray
2. https://kubernetes.io/docs/reference/kubectl/overview/
3. https://microservices-demo.github.io/

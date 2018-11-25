# Table of Contents

1. [Introduction](#introduction)
2. [OpenStack Installation](#openstack-install-basic-usage)  
  2.1. [Install and Configure Kolla-Ansible](#install-configure-kolla-ansible)  
  2.2. [Install OpenStack](#install-openstack)  
  2.3. [Prepare for Initial Use](#prepare-initial-use)  
3. [Basic OpenStack Usage](#use-openstack)  
  3.1. [Use OpenStack with OpenStack Client](#use-openstack-with-osc)  
  3.2. [Use OpenStack with Horizon](#use-openstack-with-horizon)  
4. [Next Steps](#next-steps)
5. [References](#references)


# Introduction <a name="introduction"></a>

# OpenStack Installation and Basic Usage <a name="openstack-install-basic-usage"></a>

This guide contains information about installing **OpenStack Rocky** using
Kolla-Ansible. Almost everything in this guide is taken from official
Kolla-Ansible documentation and adapted for the purposes of this workshop. [1]

All the commands listed on this guide must be executed on Jumphost unless
otherwise is noted.

## Install and Configure Kolla-Ansible <a name="install-configure-kolla-ansible"></a>

In this section, we will install Kolla-Ansible using pip and make the necessary
configuration.

```bash
sudo pip install kolla-ansible
```

Kolla-Ansible installation contains example inventory which needs to be modified
according to our environment. In order to save some time, an inventory file named
**multinode** is available in openstack directory we we can use during OpenStack
installation with Kolla-Ansible. It would be useful if you take a look at it to
see how it looks.

```bash
cd $HOME/infra-workshop/openstack
cat multinode
ansible -i multinode -m ping all
```

Kolla-Ansible installation configuration is kept in **/etc/kolla/globals.yml**.
We will use a configuration file created for the purposes of this workshop named
**globals.yml** and located in openstack directory.

Kolla-Ansible passwords are stored in **/etc/kolla/passwords.yml** but all
passwords are blank in this file so we need to fill them by using random
password generator.

```bash
sudo cp -r /usr/local/share/kolla-ansible/etc_examples/kolla /etc/
ls -al /etc/kolla
sudo cp -f globals.yml /etc/kolla/globals.yml
cat /etc/kolla/globals.yml
sudo kolla-genpwd
cat /etc/kolla/passwords.yml
```

## Install OpenStack <a name="install-openstack"></a>

We are now ready to start installing OpenStack using Kolla-Ansible.

The first thing we need to do is to bootstrap the servers so the configuration
needed by Kolla-Ansible can be done.

Bootstrapping could take up to 2 minutes.

```bash
kolla-ansible -i ./multinode bootstrap-servers
```

Once bootstrapping is done, we need to run prechecks to verify the nodes are
configured correctly.

Running prechecks could take up to 2 minutes.

```bash
kolla-ansible -i ./multinode prechecks
```

If everything went fine until here, we are now ready to execute the command
to start OpenStack installation.

The installation could take up to 30 minutes.

```bash
kolla-ansible -i ./multinode deploy
```

If you encounter issues while running the deployment, please rerun the above
command as the issues are probably temporary and the installation will probably
succeed.

## Prepare for Initial Use <a name="prepare-initial-use"></a>

Before we can start using OpenStack, we need to do few additional things such
as installing OpenStack CLI clients and generating **admin-openrc.sh** file.
It would be good to take a look at generated admin-openrc.sh file as well.

```bash
sudo pip install python-openstackclient python-glanceclient python-neutronclient
kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
cat /etc/kolla/admin-openrc.sh
```

# Basic OpenStack Usage <a name="use-openstack"></a>

## Use OpenStack with OpenStack Client <a name="use-openstack-with-osc"></a>

In this part of the workshop we will work with below OpenStack objects using
OpenStack Client (OSC). [2]

* network
* security group
* subnet
* router
* instance
* flavor
* image
* keypair

OSC accepts commands in below form

```bash
openstack [<global-options>] <object-1> <action> [<object-2>] [<command-arguments>]
```

More details regarding how to use OSC is available on [3].

Before doing anything else, we can start by listing available OpenStack Services,
users, and hypervisors.

```bash
openstack service list
openstack user list
openstack hypervisor list
```

We can also check if anything is available for use in our OpenStack instance.

```bash
openstack network list
openstack security group list
openstack subnet list
openstack router list
openstack server list
openstack flavor list
openstack image list
openstack keypair list
```

As you see there, only the default security group is available. We will now
create the rest of the resources as part of this exercise.

We start by creating an image to use for our instances we will create on
OpenStack. For this, we can use [Cirros](https://docs.openstack.org/image-guide/obtain-images.html).

```bash
cd $HOME/infra-workshop/openstack
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
openstack image create --disk-format qcow2 --container-format bare --public \
    --property os_type=linux --file cirros-0.4.0-x86_64-disk.img cirros
openstack image list
```

We then create our networks, subnets, and routers.

```bash
openstack network create --external --provider-physical-network physnet1 \
    --provider-network-type flat ext-net
openstack subnet create --no-dhcp \
    --allocation-pool start=10.0.2.150,end=10.0.2.199 --network ext-net \
    --subnet-range 10.0.2.0/24 --gateway 10.0.2.1 ext-subnet
openstack network create --provider-network-type vxlan ws-net
openstack subnet create --subnet-range 10.0.0.0/24 --network ws-net \
    --gateway 10.0.0.1 --dns-nameserver 8.8.8.8 ws-subnet
openstack router create ws-router
openstack router add subnet ws-router ws-subnet
openstack router set --external-gateway ext-net ws-router
```

We created our networks, subnets, and router. Let's look at them using OSC.

```bash
openstack network list
openstack network show ws-net
openstack subnet list
openstack subnet show ws-subnet
openstack router list
openstack router show ws-router
```

OpenStack Security Groups act as a virtual firewall for servers and other
resources on a network. It is a container for security group rules which
specify the network access rules.

We need to adjust security group access rules so we can ping our instances
and access them using SSH.

```bash
ADMIN_USER_ID=$(openstack user list | awk '/ admin / {print $2}')
ADMIN_PROJECT_ID=$(openstack project list | awk '/ admin / {print $2}')
ADMIN_SEC_GROUP=$(openstack security group list --project ${ADMIN_PROJECT_ID} | awk '/ default / {print $2}')
openstack security group rule create --ingress --ethertype IPv4 \
    --protocol icmp ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
    --protocol tcp --dst-port 22 ${ADMIN_SEC_GROUP}
openstack security group show ${ADMIN_SEC_GROUP}
```

In order for to login to our instances, we need our SSH key to be added
to authorized_keys file on the instances. We use keypairs to achieve this.

```bash
openstack keypair create --public-key ~/.ssh/id_rsa.pub ws-key
openstack keypair list
```

Before we create our first instance, we need to create flavors to use.

```bash
openstack flavor create --id 1 --ram 512 --disk 1 --vcpus 1 m1.tiny
openstack flavor create --id 2 --ram 2048 --disk 20 --vcpus 1 m1.small
openstack flavor list
```

We are now ready to create our first instance on our OpenStack!

Below command creates an instance named ws-instance1 using cirros image with
the m1.tiny flavor. It adds our public key to authorized_keys and connects it
to the network ws-net.

```bash
openstack server create \
    --image cirros \
    --flavor m1.tiny \
    --key-name ws-key \
    --network ws-net \
    ws-instance1
openstack server list
```

Once the instance is created, we can ping it and login to it via SSH from node
controller00.

```bash
ssh controller00
IPNS=$(ip netns | grep qrouter)
sudo ip netns exec $IPNS ping 10.0.0.9       # this ip should match to the ip of the instance
sudo ip netns exec $IPNS ssh cirros@10.0.0.9 # password is gocubsgo
```

## Use OpenStack via Horizon Dashboard <a name="use-openstack-with-horizon"></a>

Our installation includes **Horizon** and we can access to it via port
forwarding. Logout from jumphost and log back in using below command.
Please ensure you use the IP of your jumphost instance and point to
private key you received.

```bash
ssh -L 8089:10.1.0.11:80 ubuntu@<IP_OF_JUMPHOST> -i <PATH_TO_SSH_PRIVATE_KEY>
grep 'OS_USERNAME\|OS_PASSWORD' /etc/kolla/admin-openrc.sh
```

Open the url **http://localhost:8089** on your computer and enter the username
and password you extracted in earlier step to login to Horizon dashboard. If
you receive an error message, please enter the username and password again
until you succeed logging in.

Please expore the tabs **Compute** and **Network** where you can see
the instances, images, keypairs, networks, routers and so on.

# Next Steps <a name="next-steps"></a>

You completed the OpenStack part of the workshop. You can now move to Kubernetes
part by clicking [this link](https://github.com/fdegir/infra-workshop/tree/master/kubernetes).

# References <a name="references"></a>

1. https://docs.openstack.org/kolla-ansible/queens/user/quickstart.html
2. https://docs.openstack.org/python-openstackclient/rocky/
3. https://docs.openstack.org/python-openstackclient/rocky/cli/commands.html

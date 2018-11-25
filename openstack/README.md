# OpenStack Installation and Basic Usage

This guide contains information about installing **OpenStack Rocky** using
Kolla-Ansible. Almost everything in this guide is taken from official
Kolla-Ansible documentation and adapted for the purposes of this workshop. [1]

All the commands listed on this guide must be executed on Jumphost unless
otherwise is noted.

## Install Kolla-Ansible and Prepare the Initial Configuration

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

## Install OpenStack

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

## Prepare for Initial Use

Before we can start using OpenStack, we need to do few additional things such
as installing OpenStack CLI clients and generating **admin-openrc.sh** file.
It would be good to take a look at generated admin-openrc.sh file as well.

```bash
sudo pip install python-openstackclient python-glanceclient python-neutronclient
kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
cat /etc/kolla/admin-openrc.sh
```

## Use OpenStack

In this part of the workshop we will work with below OpenStack resources.

* network
* security group
* subnet
* router
* instance
* flavor
* image
* keypair

Before doing anything else, we can start by listing available OpenStack Services
and hypervisors.

```bash
openstack service list
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
```

We then create our networks, subnets, and routers.

```bash
openstack network create --external --provider-physical-network physnet1 \
    --provider-network-type flat ext-net
openstack subnet create --no-dhcp \
    --allocation-pool start=10.0.2.150,end=10.0.2.199 --network ext-net \
    --subnet-range 10.0.2.0/24 --gateway 10.0.2.1 ext-subnet
openstack network create --provider-network-type vxlan demo-net
openstack subnet create --subnet-range 10.0.0.0/24 --network demo-net \
    --gateway 10.0.0.1 --dns-nameserver 8.8.8.8 demo-subnet
openstack router create demo-router
openstack router add subnet demo-router demo-subnet
openstack router set --external-gateway ext-net demo-router
```




for item in {network,subnet,router,server,flavor,image,keypair}; do

openstack
openstack security group list
openstack server create --flavor m1.tiny --image cirros --network demo-net --security-group <uuid> --key-name ws-key dummy5

sudo ip netns
sudo ip netns qrouter-<uuid> ping <ip>
sudo ip netns qrouter-<uuid> ssh cirros@<ip> (gocubsgo is the password)

modify globals.yaml
kolla-ansible -i ./multinode bootstrap-servers <-- start: 02:44, end: 02:46
kolla-ansible -i ./multinode prechecks <-- start: 02:47, end: 02:49
kolla-ansible -i ./multinode deploy <-- start: 03:02, end: 03:19, restart: 03:19, reend: 03:27

while deploying, do kolla images and kolla ps on controller node




command examples
openstack service list
openstack server list
openstack image list
openstack flavor list
openstack keypair list

# References

1. https://docs.openstack.org/kolla-ansible/queens/user/quickstart.html

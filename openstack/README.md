# OpenStack Installation and Basic Command Examples

This guide contains information about installing **OpenStack** Queens version
using Kolla-Ansible. Almost everything in this guide is taken from official
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
according to our environment. In order to save some time, an inventory file is
available in infra-workshop/openstack directory which we can use during OpenStack
installation with Kolla-Ansible. It would be useful if you take a look at it to
see how it looks.

```bash
cd infra-workshop/openstack
less multinode
ansible -i multinode -m ping all
```

sudo cp -r /usr/local/share/kolla-ansible/etc_examples/kolla /etc/kolla/
Kolla-Ansible passwords are stored in **/etc/kolla/passwords.yml** but all
passwords are blank in this file so we need to fill them by using random
password generator.

```bash
sudo cp -R globals.yml /etc/kolla/globals.yml
sudo kolla-genpwd
cat /etc/kolla/password.yml
```

## Install OpenStack

We are now ready to work on OpenStack installation.

The first thing that needs to be done is to bootstrap the servers to configure
them as needed, install required packages and so on. Bootstrapping could take
up to 2 minutes.

```bash
kolla-ansible -i ./multinode bootstrap-servers
```

Once bootstrapping is done, we need to run prechecks. Running prechecks could
take up to 2 minutes.

```bash
kolla-ansible -i ./multinode prechecks
```

If everything went fine until here, we are now ready to start OpenStack
installation. The installation could take up to 30 minutes.

```bash
kolla-ansible -i ./multinode deploy
:w
```

## Use OpenStack

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

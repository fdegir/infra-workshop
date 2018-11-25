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
command as the issues are probably temporary and the installation will succeed
next time.

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

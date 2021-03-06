# Table of Contents

1. [Introduction](#introduction)
2. [OpenStack Installation](#openstack-installation)  
  2.1. [Install and Configure Kolla-Ansible](#install-configure-kolla-ansible)  
  2.2. [Install OpenStack](#install-openstack)  
  2.3. [Prepare for Initial Use](#prepare-initial-use)  
3. [Basic OpenStack Usage](#use-openstack)  
  3.1. [Use OpenStack with OpenStack Client](#use-openstack-with-osc)  
  3.2. [Use OpenStack with Horizon](#use-openstack-with-horizon)  
4. [Cleanup](#cleanup)
5. [Next Steps](#next-steps)
6. [References](#references)


# Introduction <a name="introduction"></a>

This guide contains information about installing **OpenStack** using
Kolla-Ansible and basic usage of OpenStack with OpenStack Client and
Horizon Dashboard. The version we are going to install is the latest
released version of OpenStack, **Rocky**. [1]

Information in this guide is based on official Kolla-Ansible
documentation and adapted for the purposes of this workshop. [2]

All the commands listed on this guide must be executed on jumphost unless
otherwise is noted.

# OpenStack Installation <a name="openstack-installation"></a>

This section covers the details to install and configure Kolla-Ansible,
install OpenStack, and prepare it for the initial use.

## Install and Configure Kolla-Ansible <a name="install-configure-kolla-ansible"></a>

In this section, we will install Kolla-Ansible using pip and make the necessary
configuration.

```bash
sudo pip install kolla-ansible
```

Kolla-Ansible installation contains example inventory which needs to be modified
according to our environment. In order to save some time, an inventory file named
**multinode** is available in openstack directory which we can use during OpenStack
installation with Kolla-Ansible. It would be useful to take a look at it.

```bash
cd $HOME/infra-workshop/openstack
cat multinode
ansible -i multinode -m ping all
```

Kolla-Ansible installation configuration is kept in **/etc/kolla/globals.yml**.
We will use a configuration file created for the purposes of this workshop named
**globals.yml** and located in openstack directory of the infra-workshop repo.

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
command as the issues are probably temporary and the installation will
succeed.

Once the playbook execution is completed successfully, you should
see 28 running containers on controller00 and 10 running containers
on compute00.

In order to be able to user docker on controller00 and compute00,
ubuntu user needs to be added to docker group on both nodes.

```bash
ssh controller00
sudo usermod -aG docker ubuntu
logout
ssh controller00
docker ps
CONTAINER ID        IMAGE                                                 COMMAND             CREATED              STATUS              PORTS               NAMES
67ffab96a245        kolla/ubuntu-source-horizon:rocky                     "kolla_start"       22 seconds ago       Up 21 seconds                           horizon
9d53d3890344        kolla/ubuntu-source-heat-engine:rocky                 "kolla_start"       59 seconds ago       Up 58 seconds                           heat_engine
35ee6f3ebb43        kolla/ubuntu-source-heat-api-cfn:rocky                "kolla_start"       About a minute ago   Up About a minute                       heat_api_cfn
448b092b7bdd        kolla/ubuntu-source-heat-api:rocky                    "kolla_start"       About a minute ago   Up About a minute                       heat_api
6d51554bb71d        kolla/ubuntu-source-neutron-metadata-agent:rocky      "kolla_start"       2 minutes ago        Up 2 minutes                            neutron_metadata_agent
9bb0666885f9        kolla/ubuntu-source-neutron-l3-agent:rocky            "kolla_start"       2 minutes ago        Up 2 minutes                            neutron_l3_agent
a35bc922b0c5        kolla/ubuntu-source-neutron-dhcp-agent:rocky          "kolla_start"       3 minutes ago        Up 3 minutes                            neutron_dhcp_agent
98b24eae4153        kolla/ubuntu-source-neutron-openvswitch-agent:rocky   "kolla_start"       3 minutes ago        Up 3 minutes                            neutron_openvswitch_agent
86f18aec214a        kolla/ubuntu-source-neutron-server:rocky              "kolla_start"       3 minutes ago        Up 3 minutes                            neutron_server
ee11881dda6c        kolla/ubuntu-source-openvswitch-vswitchd:rocky        "kolla_start"       8 minutes ago        Up 8 minutes                            openvswitch_vswitchd
138e4b6f8bd1        kolla/ubuntu-source-openvswitch-db-server:rocky       "kolla_start"       13 minutes ago       Up 13 minutes                           openvswitch_db
1e12963ddedf        kolla/ubuntu-source-nova-novncproxy:rocky             "kolla_start"       17 minutes ago       Up 17 minutes                           nova_novncproxy
c3396b744c36        kolla/ubuntu-source-nova-consoleauth:rocky            "kolla_start"       17 minutes ago       Up 17 minutes                           nova_consoleauth
3c3744bd8809        kolla/ubuntu-source-nova-conductor:rocky              "kolla_start"       17 minutes ago       Up 17 minutes                           nova_conductor
cdb54bc51775        kolla/ubuntu-source-nova-scheduler:rocky              "kolla_start"       17 minutes ago       Up 17 minutes                           nova_scheduler
50c20e284fab        kolla/ubuntu-source-nova-api:rocky                    "kolla_start"       17 minutes ago       Up 17 minutes                           nova_api
db0821845151        kolla/ubuntu-source-nova-placement-api:rocky          "kolla_start"       17 minutes ago       Up 17 minutes                           placement_api
0bf4dd6bf40a        kolla/ubuntu-source-glance-api:rocky                  "kolla_start"       22 minutes ago       Up 22 minutes                           glance_api
ff17ba84252d        kolla/ubuntu-source-keystone-fernet:rocky             "kolla_start"       24 minutes ago       Up 24 minutes                           keystone_fernet
4d4a91d7685c        kolla/ubuntu-source-keystone-ssh:rocky                "kolla_start"       25 minutes ago       Up 25 minutes                           keystone_ssh
d7653e9be45c        kolla/ubuntu-source-keystone:rocky                    "kolla_start"       25 minutes ago       Up 25 minutes                           keystone
c844a3b107bb        kolla/ubuntu-source-rabbitmq:rocky                    "kolla_start"       27 minutes ago       Up 27 minutes                           rabbitmq
e9241557dae8        kolla/ubuntu-source-mariadb:rocky                     "kolla_start"       27 minutes ago       Up 27 minutes                           mariadb
ac1a1a90fb56        kolla/ubuntu-source-memcached:rocky                   "kolla_start"       28 minutes ago       Up 28 minutes                           memcached
ee65e0151b8c        kolla/ubuntu-source-chrony:rocky                      "kolla_start"       29 minutes ago       Up 29 minutes                           chrony
2f5a5a2607ae        kolla/ubuntu-source-cron:rocky                        "kolla_start"       29 minutes ago       Up 29 minutes                           cron
9c301fc69f61        kolla/ubuntu-source-kolla-toolbox:rocky               "kolla_start"       29 minutes ago       Up 29 minutes                           kolla_toolbox
1184af1fd296        kolla/ubuntu-source-fluentd:rocky                     "kolla_start"       30 minutes ago       Up 30 minutes                           fluentd

ssh compute00
sudo usermod -aG docker ubuntu
logout
ssh compute00
docker ps
CONTAINER ID        IMAGE                                                 COMMAND             CREATED             STATUS              PORTS               NAMES
0d564499f14f        kolla/ubuntu-source-neutron-openvswitch-agent:rocky   "kolla_start"       4 minutes ago       Up 4 minutes                            neutron_openvswitch_agent
a9ba482e9be0        kolla/ubuntu-source-openvswitch-vswitchd:rocky        "kolla_start"       9 minutes ago       Up 9 minutes                            openvswitch_vswitchd
6bdf353e653a        kolla/ubuntu-source-openvswitch-db-server:rocky       "kolla_start"       9 minutes ago       Up 9 minutes                            openvswitch_db
b21c21eeb106        kolla/ubuntu-source-nova-compute:rocky                "kolla_start"       17 minutes ago      Up 17 minutes                           nova_compute
98b1e29e6380        kolla/ubuntu-source-nova-libvirt:rocky                "kolla_start"       18 minutes ago      Up 18 minutes                           nova_libvirt
6a2b6d06fa6b        kolla/ubuntu-source-nova-ssh:rocky                    "kolla_start"       19 minutes ago      Up 19 minutes                           nova_ssh
2340aaeffa0a        kolla/ubuntu-source-chrony:rocky                      "kolla_start"       30 minutes ago      Up 30 minutes                           chrony
3859b5a6e3aa        kolla/ubuntu-source-cron:rocky                        "kolla_start"       30 minutes ago      Up 30 minutes                           cron
7e9b3dd0ce65        kolla/ubuntu-source-kolla-toolbox:rocky               "kolla_start"       30 minutes ago      Up 30 minutes                           kolla_toolbox
a48f7d0da532        kolla/ubuntu-source-fluentd:rocky                     "kolla_start"       31 minutes ago      Up 31 minutes                           fluentd
```

## Prepare for Initial Use <a name="prepare-initial-use"></a>

Before we can start using OpenStack, we need to do few additional things such
as installing OpenStack CLI clients and generating **admin-openrc.sh** file.
It would be good to take a look at generated admin-openrc.sh file as well.

```bash
sudo pip install python-openstackclient python-glanceclient python-neutronclient
openstack service list # <-- this will fail due to missing credentials
kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
cat /etc/kolla/admin-openrc.sh
```

# Basic OpenStack Usage <a name="use-openstack"></a>

The usage instuctions on this section is pretty basic on purpose since
our aim is to exercise how to bring up the infrastructure for our purposes
You can always refer to official documentation for more details.

## Use OpenStack with OpenStack Client <a name="use-openstack-with-osc"></a>

In this part of the workshop we will work with below OpenStack objects using
OpenStack Client (OSC). [3]

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

More details regarding how to use OSC is available on [4].

Before doing anything else, we can start by listing available OpenStack services,
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
and login to them using SSH.

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
to authorized_keys file on the instances. We use keypairs to achieve this
so the our public key can be injected to instances.

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

Once the instance status becomes ACTIVE in openstack server list
command output, we can ping it and login to it via SSH from node
controller00.

```bash
ssh controller00
IPNS=$(ip netns | grep qrouter)
sudo ip netns exec $IPNS ping 10.0.0.9       # this ip should match to the ip of the instance
sudo ip netns exec $IPNS ssh cirros@10.0.0.9 # password is gocubsgo and use the right ip
```

## Use OpenStack with Horizon Dashboard <a name="use-openstack-with-horizon"></a>

Our installation includes **Horizon** and we can access to it via port
forwarding. Logout from jumphost and log back in using below command.
Please ensure you use the IP of your jumphost instance and point to
private key you received.

```bash
ssh -L 8089:10.1.0.11:80 ubuntu@<IP_OF_JUMPHOST> -i <PATH_TO_SSH_PRIVATE_KEY>
grep 'OS_USERNAME\|OS_PASSWORD' /etc/kolla/admin-openrc.sh
```

Open the url **http://localhost:8089** on your browser and enter the username
and password you extracted in earlier step to login to Horizon dashboard. If
you receive an error message, please enter the username and password again
until you succeed logging in.

Please expore the tabs **Compute** and **Network** where you can see
the instances, images, keypairs, networks, routers and so on.

Once you spent enough time on exploring the dashboard, please open **Network**
-> **Network Topology** and click **Launch Instance** from the upper right
corner of the page, creating an instance on the same network **ws-net** as
the instance we created using OSC earlier.

Since the instances are on the same network, you should be able to
ping the instances from each other and login to them using SSH.

# Cleanup <a name="cleanup"></a>

Before moving to the next part of the workshop, we need to cleanup the environment
in order to ensure the existing OpenStack installation doesn't impact
Kubernetes work.

Please execute the commands below on **jumphost**.

```bash
source /etc/kolla/admin-openrc.sh
openstack server list
openstack server delete ws-instance1 # ensure you delete all the instances you created
openstack server list
cd $HOME/infra-workshop/openstack
kolla-ansible -i ./multinode destroy --include-images --yes-i-really-really-mean-it
```

# Next Steps <a name="next-steps"></a>

You completed the OpenStack part of the workshop. You can now move to Kubernetes
part by clicking [this link](https://github.com/fdegir/infra-workshop/tree/master/kubernetes).

# References <a name="references"></a>

1. https://releases.openstack.org/rocky/index.html
2. https://docs.openstack.org/kolla-ansible/queens/user/quickstart.html
3. https://docs.openstack.org/python-openstackclient/rocky/
4. https://docs.openstack.org/python-openstackclient/rocky/cli/commands.html

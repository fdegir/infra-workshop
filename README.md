# Prepare Jumphost

This section contains the installation and configuration steps that must be
performed on Jumphost machine. The steps documented here are common for
Kolla-Ansible and Kubespray. Please copy and paste the commands to your
terminal where you are logged in to Jumphost.

## Install and Configure Docker

In order for us to install Docker on Jumphost, we need to install few
dependencies, add Docker’s official GPG key, and add Docker stable repository.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https \
    ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
sudo apt-get update
```

We can now proceed with the installation of Docker.

```bash
sudo apt-get -y install docker-ce
```

In order for us to user Docker with the user **ubuntu**, we need to add it to
**docker** group.

```bash
sudo usermod -aG docker ubuntu
```

Before using Docker, the service needs to be started. Enabling service ensures
that it starts automatically when/if the machine gets rebooted.

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

We are now ready to test if Docker installed correctly. Logout and log back in
so new group membership takes effect. Once you log back in, try running
hello-world container, which will pull down the image, run it and you should
see welcome text on your terminal.

```bash
logout
docker run hello-world
```

## Install Ansible

Both [Kolla Ansible](https://docs.openstack.org/kolla-ansible/queens/index.html)
and [Kubespray](https://github.com/kubernetes-incubator/kubespray) use
[Ansible](https://www.ansible.com/) for installation and configuration of
OpenStack and Kubernetes respectively.

This section contains instructions to install and configure Ansible version
2.5.8 on Jumphost which is common for both Kolla Ansible and Kubespray.

We first install the packages required by Ansible.

```bash
sudo apt-get install -y python-pip python-dev libffi-dev gcc \
    libssl-dev python-selinux
sudo pip install -U pip==9.0.3
```

We can now install Ansible.

```bash
sudo pip install ansible==2.5.8
```

Please use below commands which create the file named **inventory** with the
details of target nodes we will be using in ubuntu user's home directory. We
also need to install python on target nodes to ensure we don't have trouble
using Ansible against those nodes later on.

```bash
cat <<EOF > $HOME/inventory
[all]
jumphost   ansible_connection=local ansible_user=ubuntu ansible_become=true
controller ansible_connection=ssh ansible_user=ubuntu ansible_become=true
compute    ansible_connection=ssh ansible_user=ubuntu ansible_become=true
EOF
cat $HOME/inventory
ansible -i $HOME/inventory all -m raw -a "apt-get install -y python-dev"
```

Once this is done, you can now use Ansible against the nodes you declared in
your inventory.

```bash
ansible -i $HOME/inventory all -a "uname -an"
```

## Next Steps

If everything went fine until this point, it means you are now ready to
move to **OpenStack** part of the workshop. Please open the instructions by
clicking [this link](https://github.com/fdegir/infra-workshop/tree/master/openstack).

# References
1. https://docs.docker.com/install/linux/docker-ce/ubuntu/
2. https://docs.openstack.org/kolla-ansible/queens/user/quickstart.html

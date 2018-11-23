# Prepare Jumphost
This section contains the installation and configuration steps that must be performed on Jumphost machine.
The steps documented here are common for Kolla-Ansible and Kubespray. Please copy and paste the commands on your terminal where you logged in to Jumphost.

## Install and Configure Docker
In order for us to install Docker on Jumphost, we need to install few dependencies, add Docker’s official GPG key, and add Docker stable repository.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
```

We can now proceed with the installation of Docker.

```bash
sudo apt-get -y install docker-ce
```

In order for us to user Docker with the user **ubuntu**, we need to add it to **docker** group.

```bash
sudo usermod -aG docker ubuntu
```

Before using Docker, the service needs to be started. Enabling service ensures that it starts automatically when/if the machine gets rebooted.

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

We are now ready to test if Docker installed correctly. Logout and log back in so new group membership takes effect. Once you log back in, try running hello-world container, which will pull down the image, run it and you should see welcome text on your terminal.

```bash
logout
docker run hello-world
```

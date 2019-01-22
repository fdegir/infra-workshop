# Jumphost Preparation for OpenStack Healthcheck

Before issuing before commands, please make sure admin-openrc.sh file
is copied to jumphost.

Apart from making admin-openrc.sh file available, the env file created
in below command should be adjusted according to deployment.

```bash
cat <<EOF > $HOME/env
INSTALLER_IP=<ip to controller>
TEST_DB_URL=http://testresults.opnfv.org/test/api/v1/results
ENERGY_RECORDER_API_URL=http://energy.opnfv.fr/resources
EXTERNAL_NETWORK=<external network created on the deployment>
INSTALLER_TYPE=kolla
CI_LOOP=daily
BUILD_TAG=notag
NODE_NAME=noname
FUNCTEST_MODE=tier
FUNCTEST_SUITE_NAME=healthcheck
FUNCTEST_VERSION=latest
DEPLOY_SCENARIO=os-nosdn-nofeature-noha
EOF
```

We can now continue with the rest of the preparation

```bash
sudo apt install -y less wget xz-utils
mkdir $HOME/functest-results
mkdir $HOME/images && cd $HOME/images
wget -q http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
cd $HOME
wget -qO- https://get.docker.com/ | sh
```

Now we can run healthcheck suite.

```bash
sudo docker run --env-file env \
  -v $(pwd)/admin-openrc.sh:/home/opnfv/functest/conf/env_file \
  -v $(pwd)/images:/home/opnfv/functest/images \
  -v $(pwd)/functest-results:/home/opnfv/functest/results \
  opnfv/functest-healthcheck:latest
```

# OpenStack Healthcheck

Before issuing below commands, please make sure OpenStack admin-openrc.sh
is copied to jumphost.

Apart from making admin-openrc.sh file available, the env file created
in below command should be adjusted according to deployment.

```bash
cat <<EOF > $HOME/env
INSTALLER_IP=<ip to controller>
TEST_DB_URL=http://testresults.opnfv.org/test/api/v1/results
ENERGY_RECORDER_API_URL=http://energy.opnfv.fr/resources
EXTERNAL_NETWORK=<external network created on the deployment>
INSTALLER_TYPE=<installer>
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

Now we can check our access to OpenStack followed by running healthcheck suite.

```bash
source admin-openrc.sh
openstack service list
sudo docker run --env-file env \
  -v $(pwd)/admin-openrc.sh:/home/opnfv/functest/conf/env_file \
  -v $(pwd)/images:/home/opnfv/functest/images \
  -v $(pwd)/functest-results:/home/opnfv/functest/results \
  opnfv/functest-healthcheck:latest
```

Once the test finishes, you should be presented with a report showing what
happened with the test cases.

```bash
+----------------------------+------------------+---------------------+------------------+----------------+
|         TEST CASE          |     PROJECT      |         TIER        |     DURATION     |     RESULT     |
+----------------------------+------------------+---------------------+------------------+----------------+
|      connection_check      |     functest     |     healthcheck     |      00:01       |      PASS      |
|       tenantnetwork1       |     functest     |     healthcheck     |      00:03       |      PASS      |
|       tenantnetwork2       |     functest     |     healthcheck     |      00:04       |      PASS      |
|          vmready1          |     functest     |     healthcheck     |      00:04       |      PASS      |
|          vmready2          |     functest     |     healthcheck     |      00:05       |      PASS      |
|         singlevm1          |     functest     |     healthcheck     |      00:35       |      PASS      |
|         singlevm2          |     functest     |     healthcheck     |      00:41       |      PASS      |
|         vping_ssh          |     functest     |     healthcheck     |      00:39       |      PASS      |
|       vping_userdata       |     functest     |     healthcheck     |      00:39       |      PASS      |
|        cinder_test         |     functest     |     healthcheck     |      00:32       |      PASS      |
|         api_check          |     functest     |     healthcheck     |      00:00       |      SKIP      |
|     snaps_health_check     |     functest     |     healthcheck     |      00:00       |      SKIP      |
|            odl             |     functest     |     healthcheck     |      00:00       |      SKIP      |
+----------------------------+------------------+---------------------+------------------+----------------+
```

Apart from running all tests cases that are part of Functest Healthcheck suite,
it is possible to run single test cases. Here is how you can do it.

```bash
sudo docker run -it --env-file env \
  -v $(pwd)/admin-openrc.sh:/home/opnfv/functest/conf/env_file \
  -v $(pwd)/images:/home/opnfv/functest/images \
  -v $(pwd)/functest-results:/home/opnfv/functest/results \
  opnfv/functest-healthcheck:latest bash
run_tests -t <test_case_name_you_choose_from_the_list_above>
```

# Kubernetes  Healthcheck

Before issuing below commands, please make sure Kubernetes kube config
is copied to jumphost.

Apart from making kube config file available, the env file created
in below command should be adjusted according to deployment.

```bash
cat <<EOF > $HOME/env
INSTALLER_IP=<ip to master>
TEST_DB_URL=http://testresults.opnfv.org/test/api/v1/results
ENERGY_RECORDER_API_URL=http://energy.opnfv.fr/resources
INSTALLER_TYPE=<installer>
CI_LOOP=daily
BUILD_TAG=notag
NODE_NAME=noname
FUNCTEST_MODE=tier
FUNCTEST_SUITE_NAME=healthcheck
FUNCTEST_VERSION=latest
DEPLOY_SCENARIO=k8-flannel-nofeature-noha
EOF
```

We can now continue with the rest of the preparation

```bash
sudo apt install -y less wget xz-utils
mkdir $HOME/functest-results
cd $HOME
wget -qO- https://get.docker.com/ | sh
KUBE_MASTER_URL=$(grep -r server ~/.kube/config | awk '{print $2}')
KUBE_MASTER_IP=$(echo $KUBE_MASTER_URL | awk -F "[:/]" '{print $4}')
cat << EOF > ~/k8s.creds
KUBERNETES_PROVIDER=local
KUBE_MASTER_URL=$KUBE_MASTER_URL
KUBE_MASTER_IP=$KUBE_MASTER_IP
EOF
```

Now we can check our access to Kubernetes followed by running healthcheck suite.

```bash
kubectl get pod --all-namespaces
sudo docker run --env-file env \
  -v $(pwd)/k8s.creds:/home/opnfv/functest/conf/env_file \
  -v $(pwd)/.kube/config:/root/.kube/config \
  -v $(pwd)/functest-results:/home/opnfv/functest/results \
  opnfv/functest-kubernetes-healthcheck:latest
```



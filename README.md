# Create a Kubernetes cluster with Kubespray

The `kubespray` pack is located here: https://github.com/kubernetes-sigs/kubespray

## Prerequisites

You need to have installed on the host machine few packages that are available via Python-pip. The following command runs in kubespray directory:

```shell
pip install -r src/requirements.txt
```

## Configuration

### Modify ansible.cfg

File location: `src/ansible.cfg`

Add/modify following:

- pipelining
- log_path
- provilege_escalation section

```ini
[ssh_connection]
pipelining=False

[defaults]
log_path = ./ansible.log

[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
```

### Modify Vagrantfile

File location: `src/Vagrantfile`

In order to generate in Virtualbox the cluster you need to update few parameters:

- number of nodes (e.g. 3, odd number to comply with the needs of etcd)
- name prefix (e.g. "k8s" it will create k8s-1, k8s-2, k8s-3, etc.)
- memory (e.g. 4GB)
- number of cpu's (e.g. 2 cpu)
- subnet (e.g. "192.168.10"; it will create the ip's 101, 102, 103,  etc.)
- os (e.g. "ubuntu1604", according to the keys in SUPPORTED_OS)

I chose the following values that created 

```ruby
$num_instances = 3
$instance_name_prefix = "k8s"
$vm_memory = 4096
$vm_cpus = 2
$subnet = "192.168.10"
$os = "ubuntu1604"
```

### Modify inventory.ini

File location: `src/inventory/sample/inventory.ini`

In order to specify the role of each node you need to modify several sections:

#### Section "all servers"

If you chose 3 instances In Vagrantfile then for node[1-3] should be specified the ip addresses according to the subnet in Vagrantfile like below:

```ini
[all]
node1 ansible_host=192.168.10.101 ip=192.168.10.101 etcd_member_name=etcd1
node2 ansible_host=192.168.10.102 ip=192.168.10.102 etcd_member_name=etcd2
node3 ansible_host=192.168.10.103 ip=192.168.10.103 etcd_member_name=etcd3
```

#### Section "kube-master"

Choose one master node out of the 3 nodes of the cluster:

```ini
[kube-master]
node1
```

#### Section "etcd"

Choose an odd number (2k+1) of nodes where `etcd` will run:

```ini
[etcd]
node1
node2
node3
```

#### Section "kube-node"

Choose the worker nodes. They may be separate from the master nodes or master nodes may be also worker (but with less resources)

```ini
[kube-node]
node2
node3
```

Another cluster architecture may be with 4 nodes:

- 4 nodes total (e.g. node[1-4]) out of which:
  - 1 master node (e.g. node1)
  - 3 ectd nodes (e.g. node[1-3])
  - 3 workers (e.g. node[2-4])

Another cluster architecture may be with 5 nodes:

- 5 nodes total (e.g. node[1-5]) out of which:
  - 2 master nodes (e.g. node[1-2])
  - 5 ectd nodes (e.g. node[1-5])
  - 3 workers (e.g. node[3-5])

### Proxy

File location: `src/inventory/sample/group_vars/all/all.yml`

If you run the cluster behind a proxy then you must specify this. You may need to modify the following assuming your proxy is `http://192.168.56.200:3128`:

```yaml
http_proxy: "http://192.168.56.200:3128"
https_proxy: "http://192.168.56.200:3128"
```

You may also need to change the `no_proxy` parameter.

### Enable several useful addons in your cluster

File location: `src/inventory/sample/group_vars/k8s-cluster/addons.yml`

#### Enable Kubernetes Dashboard

In order to be available to use the Dashboard you need to change the parameter:

```yaml
dashboard_enabled: true
```

#### Enable Nginx Ingress

In order to make available the services deployed in your Kubespray cluster you need to deploy the Nginx Ingress.

```yaml
ingress_nginx_enabled: true
```

#### Enable Metrics Server

In order to get metrics from the cluster you need to change the parameter:

```yaml
metrics_server_enabled: true
```

### Set various parameters for your cluster

File location: `src/inventory/sample/group_vars/k8s-cluster/k8s-cluster.yml`

#### Choose the network plugin

You may choose between several plugins like: calico, flannel and few other. The default is `calico`.

```yaml
kube_network_plugin: calico
```

According to the network plugin chosenyou may want to update specific parameters in the corresponding config file.

For `calico` you need to modify the file `src/inventory/sample/group_vars/k8s-cluster/k8s-net-calico.yml`

#### Choose the cluster name

This is the suffix of the services.

```yaml
cluster_name: cluster.local
```

#### Choose the internal IP address blocks for pods and services

This subnets *must be unused* in your network.

```yaml
kube_service_addresses: 10.233.0.0/18
kube_pods_subnet: 10.233.64.0/18
```

## Create cluster

There are two options to create the cluster:

### Create the cluster using Vagrant

This option will:

- create the servers in Virtualbox
- start the installation of cluster components on each node according to the configuration files modified previously

In this case you simply run the following command that will create the servers in Virtualbox and install the Kubernets components:

```shell
vagrant up
```

### Create the cluster on existing servers

This option assumes:

- the servers are already created
- the servers have operating system compliant with Kubernetes packages (see SUPPORTED_OS in kubespray/Vagrantfile)
- the inventory file is modified considering ip addresses already alocated to the servers
- Python is already installed on all servers as the installation is done with Ansible

In this case you run the following command that will install the Kubernets components:

```shell
ansible-playbook -i inventory/sample/inventory.ini cluster.yml
```

## Additional configuration

### Install Chrony on all cluster nodes

Login to each cluster node and execute the commands:

```shell
sudo su -
apt update && apt install chrony -y
systemctl start chrony
systemctl enable chrony
timedatectl
```

### Update Nginx Ingress Controller

Edit the DaemonSet `ingress-nginx-controller` and add the following argument to the element `args` of container specification:

```yaml
--report-node-internal-ip-address
```

This option will help to display for each ingress the IP address of the endpoint.

## Troubleshooting

### Installation stuck during container image download

You may need to add the user that connects to cluster nodes via Ansible to group `docker`.

If you connect with vagrant user then apply the following, otherwise change the username:

```shell
sudo usermod -aG docker vagrant
```

### Remove proxy

If you specified the http_proxy and https_proxy but you want to modify, you need to:

- login to each cluster node, either master or worker node
- edit the file `/etc/systemd/system/docker.service.d/http-proxy.conf`
- restart docker service

If you want to remove the proxy configuration then you should delete the above mentioned file on each cluster node.

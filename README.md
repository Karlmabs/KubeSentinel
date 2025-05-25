
# KubeSentinel: Automated Kubernetes Cluster with Integrated Monitoring

KubeSentinel is a fully automated Kubernetes cluster setup with integrated monitoring capabilities. The system is tested on:

- Windows
- Ubuntu Desktop
- Mac Intel-based systems

If you are a MAC Silicon user, you may need to adapt the configuration for ARM architecture.

## Cluster Architecture

KubeSentinel deploys a complete Kubernetes environment with:

- 1 Control Plane Node (controlplane)
- 2 Worker Nodes (node01, node02)
- 1 Monitoring Node running Grafana

## Setup Prerequisites

- A working Vagrant setup using Vagrant + VirtualBox
- 8 GB+ RAM workstation (VMs use 3 vCPUs and 4+ GB RAM)


## Documentation

The cluster is deployed with the latest stable Kubernetes version.


## System Requirements

1. Working Vagrant setup with VirtualBox
2. 8 GB+ RAM workstation as the VMs use 3 vCPUs and 4+ GB RAM

## For MAC/Linux Users

The latest version of Virtualbox for Mac/Linux can cause issues.

Create/edit the /etc/vbox/networks.conf file and add the following to avoid any network-related issues.
<pre>* 0.0.0.0/0 ::/0</pre>

or run below commands

```shell
sudo mkdir -p /etc/vbox/
echo "* 0.0.0.0/0 ::/0" | sudo tee -a /etc/vbox/networks.conf
```

So that the host only networks can be in any range, not just 192.168.56.0/21 as described here:
https://discuss.hashicorp.com/t/vagrant-2-2-18-osx-11-6-cannot-create-private-network/30984/23

## Bring Up the Cluster

To provision the cluster, execute the following commands.

```shell
git clone https://github.com/YourUsername/KubeSentinel.git
cd KubeSentinel
vagrant up
```
## Set Kubeconfig file variable

```shell
cd KubeSentinel
cd configs
export KUBECONFIG=$(pwd)/config
```

or you can copy the config file to .kube directory.

```shell
cp config ~/.kube/
```

## Kubernetes Dashboard

The Kubernetes Dashboard is automatically installed as part of the cluster setup.

## Kubernetes Dashboard Access

To get the login token, copy it from _config/token_ or run the following command:
```shell
kubectl -n kubernetes-dashboard get secret/admin-user -o go-template="{{.data.token | base64decode}}"
```

Make the dashboard accessible:
```shell
kubectl proxy
```

Open the site in your browser:
```shell
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
```

## Grafana Monitoring

KubeSentinel includes a dedicated monitoring node running Grafana for cluster metrics visualization.

Access Grafana at:
```
http://10.0.0.13:3000
```

Default login credentials:
- Username: admin
- Password: admin

Upon first login, you'll be prompted to change the default password.

## To shutdown the cluster,

```shell
vagrant halt
```

## To restart the cluster,

```shell
vagrant up
```

## To destroy the cluster,

```shell
vagrant destroy -f
```
# Network Architecture

```
                  +-------------------+
                  |    External       |
                  |  Network/Internet |
                  +-------------------+
                           |
                           |
             +-------------+--------------+
             |        Host Machine        |
             |     (Internet Connection)  |
             +-------------+--------------+
                           |
                           | NAT
             +-------------+--------------+
             |        NAT Network         |
             |        10.0.0.0/24         |
             +-------------+--------------+
                           |
                           |
       +--------+----------+-----------+----------+
       |        |          |           |          |
+------+--+ +---+----+ +---+----+ +----+------+  |
|  Control | | Worker | | Worker | | Monitoring|  |
|   Plane  | | Node01 | | Node02 | |   Node    |  |
| 10.0.0.10| |10.0.0.11| |10.0.0.12| | 10.0.0.13 |  |
+----------+ +--------+ +--------+ +-----------+  |
       |        |          |           |          |
       +--------+----------+-----------+----------+
```

This network architecture shows:

1. The host machine connected to the external network/internet
2. The NAT network providing connectivity between the host and all nodes
3. Four nodes in the cluster:
   - Control Plane (10.0.0.10): Kubernetes control plane node
   - Worker Node01 (10.0.0.11): Kubernetes worker node
   - Worker Node02 (10.0.0.12): Kubernetes worker node
   - Monitoring Node (10.0.0.13): Dedicated node running Grafana for monitoring

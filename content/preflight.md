# Setup OpenShift Container Platform
In this section we are going to setup a simple OpenShift based on a single Master architecture and three compute nodes. The Linux platform used for this tutorial is RHEL7. Setup of OpenShift on **Atomic Host** is not described in this tutorial. We assume the **OpenShift** platform subscriptions are available. The setup is based on Virtual Machines meeting requirements as stated in product documentation.

## Architecture
This tutorial assumes the following architecture. There are four machines:

* **Master:** master.openshift.com
* **NodeA:** nodea.openshift.com
* **NodeB:** nodeb.openshift.com
* **NodeC:** nodec.openshift.com

The **Master** is the scheduler/orchestrator of the cluster and the API endpoint. It also host a **Router** for the applications hosted on the nodes. The nodes are also called the **Compute Nodes**.

Here the setup we're going to bring-up:
![](../images/setup.png?raw=true)

The majority of storage requirements are related to Docker storage. Each VM is equipped with an additional disk, e.g. ``/dev/sdb`` for LVM device mapper as loop devices are not supported in a production. An additional VM for persistent storage is used. We assume NFS Server and iSCSI server available on that machine. Please, note this machine is not part of the OpenShift envinronment and it is only for external storage.

All of the VMs should be on the same logical network and be able to access one another resolving DNS entries. The setup requires DNS name resolving via an external DNS server for all the VMs.
```
master.openshift.com    A   10.10.10.19
nodea.openshift.com     A   10.10.10.13
nodeb.openshift.com     A   10.10.10.17
nodec.openshift.com     A   10.10.10.22
storage.openshift.com   A   10.10.10.21
```

Resolving hostnames only via ``/etc/hosts`` file is not enough. There an appendix section in this tutorial on configuring DNSmasq for the OpenShift requirements. Remember that the NetworkManager may make changes the DNS configuration and resolver. Properly configure interfaces' DNS settings and/or configure NetworkManager appropriately.

Also configure a DNS entry to route all the user application to the Master node. This is accomplished by configuring a wildcard mask on the DNS server. For example, create a wildcard DNS entry for cloud applications that has a low time-to-live value (TTL) and points to the public IP address of the host where the router will be deployed
```
*.cloud.openshift.com. 300 IN  A 10.10.10.19
```

Some sections of this tutorial refer to an high-availability setup with three master for fully redundancy of the control plane and routing layer. Here the ha-setup

![](../images/ha-setup.png?raw=true)

The ha-setup requires DNS name resolving via an external DNS server for all the VMs.
```
master01.openshift.com    A   10.10.10.10
master02.openshift.com    A   10.10.10.16
master03.openshift.com    A   10.10.10.18
node01.openshift.com     A   10.10.10.12
node02.openshift.com     A   10.10.10.14
node03.openshift.com     A   10.10.10.25
storage.openshift.com   A   10.10.10.21
```

DNS wildcard mask will point to the public IP address of the load balancer
```
*.cloud.openshift.com. 300 IN  A 10.10.10.11
```

## Security
OpenShift platform relies on Security-Enhanced Linux to work properly. Selinux must be enabled, i.e. set to "enforcing" on all of the servers before installing OpenShift Container Platform or the installer will fail.

The OpenShift installation automatically creates a set of internal firewall rules on each host using iptables. However, if the network configuration uses an external firewall, we should ensure infrastructure components can communicate with each other through specific ports that act as communication endpoints for certain processes or services. Plaese, refer to product documentation for a complete list of ports used by all the component of the platform.

## Install Docker
On all the OpenShift hosts, enable the proper subscriptions and install the setup utilities
```
# yum install -y atomic-openshift-utils
```

Install Docker on all master and node hosts and configure the Docker storage options before installing OpenShift Container Platform.
```
# yum install -y docker
# vi /etc/sysconfig/docker-storage-setup
DEVS=/dev/sdb
VG=docker

# docker-storage-setup
# systemctl star docker
# systemctl enable docker

# lsblk
NAME                          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
...
sdb                             8:16   0   20G  0 disk
└─sdb1                          8:17   0   20G  0 part
  ├─docker-docker--pool_tmeta 253:2    0   24M  0 lvm
  │ └─docker-docker--pool     253:4    0    8G  0 lvm
  └─docker-docker--pool_tdata 253:3    0    8G  0 lvm
    └─docker-docker--pool     253:4    0    8G  0 lvm
...
```

## Host Access
The OpenShift installation method is based on **Ansibe** playbook. It requirer a user that has access to all hosts. To run the installer as a non-root user, passwordless sudo rights must be configured on each destination host.

Generate an SSH key on the host where you will invoke the installation process. This can be the master host.
```
[root@master ~]# ssh-keygen
```

Do not use a password. An easy way to distribute SSH keys is by using a bash loop:
```
[root@master ~]# for host in \
    master.openshift.com \
    nodea.openshift.com \
    nodeb.openshift.com \
    nodec.openshift.com; \
do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
done
```

## Install OpenShift
The OpenShift installation method is based on **Ansibe** playbook. The ``/etc/ansible/hosts`` file is the Ansible inventory file for the playbook to use during the installation. The inventory file describes the configuration of the OpenShift Container Platform cluster.
```
[root@master ~]# cat /etc/ansible/hosts
# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
# SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=root

deployment_type=openshift-enterprise
containerized=false
os_sdn_network_plugin_name='redhat/openshift-ovs-subnet'
openshift_router_selector='region=infra'
openshift_registry_selector='region=infra'

# uncomment the following to enable htpasswd authentication
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/htpasswd'}]

# host group for masters
[masters]
master.openshift.com

# host group for nodes, includes region info
[nodes]
master.openshift.com openshift_node_labels="{'region': 'infra', 'zone': 'default'}" openshift_schedulable=true
nodea.openshift.com openshift_node_labels="{'region': 'primary', 'zone': 'east'}"
nodeb.openshift.com openshift_node_labels="{'region': 'primary', 'zone': 'west'}"
# nodec.openshift.com openshift_node_labels="{'region': 'primary', 'zone': 'west'}"
```

Please, note that we are escluding ``nodec.openshift.com`` from the cluster since we'll add later to demonstrate hot to scale the cluster.

Make sure Ansible is able to reach all nodes
```
[root@master ~]# ansible all -m ping
nodea.openshift.com | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
nodeb.openshift.com | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
master.openshift.com | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Start the installation of the OpenShift Platform
```
[root@master ~]# ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/config.yml
```

This will take some time. Once the istallation completes, check the status of the cluster
```
[root@master ~]# oc get nodes
NAME                          STATUS    AGE
master.openshift.com   Ready     1h
nodea.openshift.com    Ready     1h
nodeb.openshift.com    Ready     1h
```

In case of something went wrong, uninstall the platform if required
```
[root@master ~]# ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/adhoc/uninstall.yml
```

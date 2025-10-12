### Hybrid Infra Management Tool - kcli 
#### Example: Red Hat OpenStack On Red Hat OpenShift  on KVM Infra Provider via kcli 
- **NOTE: kcli plan can be used for other cloud infra provider**
#### Baremetal Node Information | Hypervisor Host (TESTED):
* Dell R440 | 40 Core Cpu | 256 GB Ram
* HOST OS: RHEL 9.4 
* Reference: 
  * <https://github.com/karmab/kcli>
  * <https://kcli.readthedocs.io/en/latest/>
* **NOTE: Tested using root permisions only.**
#### Register RHEL Node:
```
subscription-manager register --username _RHN_USERNAME_
```
#### Update the RHEL node with latest updates, basic utility packages & ssh key gneration: 
```
dnf update -y
dnf install tmux mc podman-docker bash-completion vim jq tar git yum-utils  -y
ssh-keygen -t ed25519
```
#### Install libvirt (KVM Virtualization):
```
dnf -y install libvirt libvirt-daemon-driver-qemu qemu-kvm
usermod -aG qemu,libvirt $(id -un)
newgrp libvirt
systemctl enable --now libvirtd
systemctl status libvirtd
```
#### Install Local NTP Server:
```
dnf install chrony -y
```
#### Configure NTP Server (chrony):
```
cat <<EOF> /etc/chrony.conf
server 0.rhel.pool.ntp.org iburst
server 1.rhel.pool.ntp.org iburst
server 2.rhel.pool.ntp.org iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony
bindcmdaddress ::
allow all
EOF
```
#### Enable Local NTP Service:
```
systemctl enable --now chronyd
systemctl status chronyd
chronyc tracking
```
#### Install [kcli](https://kcli.readthedocs.io/en/latest/) - Hybrid Infra Management Toolchain:
```
ssh-keygen -t ed25519  # Ignore if already created root SSH key pair
dnf -y copr enable karmab/kcli
dnf -y install kcli
```
#### Install & Eanble ksushy service o query vms with redfish APIs
```
kcli create sushy-service
systemctl status ksushy
```
#### Configure default pool for kcli:
```
kcli create pool -p /var/lib/libvirt/images default 
```
#### Check if reboot requires or not
```
needs-restarting -r
```
- **NOTE: # More information: https://access.redhat.com/solutions/27943**

#### Create Bastion Host for Red Hat OpenStack & Red Hat OpenShift
```
kcli create plan -f infra-rhoso-cluster.redhat.lab.kcli rhoso-cluster -A
```
#### Create Local Registry on Bastion Host
```
kcli ssh bastion-rhoso-cluster.redhat.lab
sudo /root/MakeMyLocalRegistry.Bash
```
#### Download Red Hat OpenShift Container Platform (RHOCP) Pull Secret To openshif-pull-secret.json:
```
https://console.redhat.com/openshift/install/pull-secret
```
#### Install oc/kubectl/opm CLIs on Hypervisor Host:
```
kcli download oc -P version=stable -P tag='4.16'
kcli download kubectl -P tag='4.16'
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/opm-linux.tar.gz
tar -zxvf opm-linux.tar.gz
cp kubectl oc opm /usr/bin/
```

#### Create Red Hat OpenShift Cluster (RHOCP) at Red Hat Lab (redhat.lab) From Hypervisor Host :
```
kcli create cluster openshift --pf rhoso-cluster.redhat.lab.kcli -P tag="4.16" -P plan="rhoso-cluster" -P cluster="rhoso-cluster" -P pool="rhoso-cluster"
```
- **NOTE: tag="4.16"  will decide your Red Hat OpenShift Version**

### Sample Expected Output From Hypervisor Host For Red Hat Openshift Version 4.16 

```
$ export KUBECONFIG=/root/.kcli/clusters/rhoso-cluster/auth/kubeconfig
$ oc get nodes
NAME                                  STATUS   ROLES                  AGE    VERSION
rhoso-cluster-ctlplane-0.redhat.lab   Ready    control-plane,master   143m   v1.29.8+f10c92d
rhoso-cluster-ctlplane-1.redhat.lab   Ready    control-plane,master   143m   v1.29.8+f10c92d
rhoso-cluster-ctlplane-2.redhat.lab   Ready    control-plane,master   143m   v1.29.8+f10c92d
rhoso-cluster-worker-0.redhat.lab     Ready    worker                 130m   v1.29.8+f10c92d
rhoso-cluster-worker-1.redhat.lab     Ready    worker                 129m   v1.29.8+f10c92d
rhoso-cluster-worker-2.redhat.lab     Ready    worker                 129m   v1.29.8+f10c92d
```


### Sample Expected Output From Hypervisor Host From kcli
```
kcli list network && kcli list dns redhat.lab && kcli list vms && cat /etc/hosts
Listing Networks...
+--------------+--------+------------------+------+------------+------+
| Network      |  Type  |       Cidr       | Dhcp |   Domain   | Mode |
+--------------+--------+------------------+------+------------+------+
| RedHatLabNet | routed |  10.10.10.0/24   | True | redhat.lab | nat  |
| default      | routed | 192.168.122.0/24 | True |  default   | nat  |
+--------------+--------+------------------+------+------------+------+
+-------------------------------------+------+-----+----------------------------+
|                Entry                | Type | TTL |            Data            |
+-------------------------------------+------+-----+----------------------------+
|   bastion-rhoso-cluster.redhat.lab  |  A   |  0  | 10.10.10.10 (RedHatLabNet) |
|     dns-rhoso-cluster.redhat.lab    |  A   |  0  | 10.10.10.1 (RedHatLabNet)  |
| rhoso-cluster-ctlplane-0.redhat.lab |  A   |  0  | 10.10.10.12 (RedHatLabNet) |
| rhoso-cluster-ctlplane-1.redhat.lab |  A   |  0  | 10.10.10.13 (RedHatLabNet) |
| rhoso-cluster-ctlplane-2.redhat.lab |  A   |  0  | 10.10.10.14 (RedHatLabNet) |
|  rhoso-cluster-worker-0.redhat.lab  |  A   |  0  | 10.10.10.15 (RedHatLabNet) |
|  rhoso-cluster-worker-1.redhat.lab  |  A   |  0  | 10.10.10.16 (RedHatLabNet) |
|  rhoso-cluster-worker-2.redhat.lab  |  A   |  0  | 10.10.10.17 (RedHatLabNet) |
+-------------------------------------+------+-----+----------------------------+
+--------------------------+--------+-------------+----------------------------------------------------+---------------+-----------------+
|           Name           | Status |      Ip     |                       Source                       |      Plan     |     Profile     |
+--------------------------+--------+-------------+----------------------------------------------------+---------------+-----------------+
|  bastion-rhoso-cluster   |   up   | 10.10.10.10 |                       rhel94                       | rhoso-cluster | rhel94-c4m8d100 |
| rhoso-cluster-ctlplane-0 |   up   | 10.10.10.12 | rhcos-416.94.202406251923-0-openstack.x86_64.qcow2 | rhoso-cluster |      kvirt      |
| rhoso-cluster-ctlplane-1 |   up   | 10.10.10.13 | rhcos-416.94.202406251923-0-openstack.x86_64.qcow2 | rhoso-cluster |      kvirt      |
| rhoso-cluster-ctlplane-2 |   up   | 10.10.10.14 | rhcos-416.94.202406251923-0-openstack.x86_64.qcow2 | rhoso-cluster |      kvirt      |
|  rhoso-cluster-worker-0  |   up   | 10.10.10.15 | rhcos-416.94.202406251923-0-openstack.x86_64.qcow2 | rhoso-cluster |      kvirt      |
|  rhoso-cluster-worker-1  |   up   | 10.10.10.16 | rhcos-416.94.202406251923-0-openstack.x86_64.qcow2 | rhoso-cluster |      kvirt      |
|  rhoso-cluster-worker-2  |   up   | 10.10.10.17 | rhcos-416.94.202406251923-0-openstack.x86_64.qcow2 | rhoso-cluster |      kvirt      |
+--------------------------+--------+-------------+----------------------------------------------------+---------------+-----------------+
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.10.10.11 api.rhoso-cluster.redhat.lab
10.10.10.254 console-openshift-console.apps.rhoso-cluster.redhat.lab oauth-openshift.apps.rhoso-cluster.redhat.lab prometheus-k8s-openshift-monitoring.apps.rhoso-cluster.redhat.lab
10.10.10.10 bastion-rhoso-cluster bastion-rhoso-cluster.RedHatLabNet bastion-rhoso-cluster.redhat.lab # KVIRT
10.10.10.12 rhoso-cluster-ctlplane-0 rhoso-cluster-ctlplane-0.RedHatLabNet rhoso-cluster-ctlplane-0.redhat.lab # KVIRT
10.10.10.13 rhoso-cluster-ctlplane-1 rhoso-cluster-ctlplane-1.RedHatLabNet rhoso-cluster-ctlplane-1.redhat.lab # KVIRT
10.10.10.14 rhoso-cluster-ctlplane-2 rhoso-cluster-ctlplane-2.RedHatLabNet rhoso-cluster-ctlplane-2.redhat.lab # KVIRT
10.10.10.15 rhoso-cluster-worker-0 rhoso-cluster-worker-0.RedHatLabNet rhoso-cluster-worker-0.redhat.lab # KVIRT
10.10.10.16 rhoso-cluster-worker-1 rhoso-cluster-worker-1.RedHatLabNet rhoso-cluster-worker-1.redhat.lab # KVIRT
10.10.10.17 rhoso-cluster-worker-2 rhoso-cluster-worker-2.RedHatLabNet rhoso-cluster-worker-2.redhat.lab # KVIRT
```

#### Follow Official Beta Document Deploying Red Hat OpenStack Services on OpenShift: 
```
https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html-single/deploying_red_hat_openstack_services_on_openshift/index
```


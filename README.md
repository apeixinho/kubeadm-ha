# kubeadm-highavailiability (English / Chinese) - kubeadm-based kubernetes high availability cluster deployment, support v1.14.x v1.11.x v1.9.x v1.7.x v1.6.x version

![k8s logo](images/kubernetes.png)

- [Chinese documentation (for v1.14.x version)] (README_CN.md)
- [English document(for v1.14.x version)](README.md)
- [Chinese documentation (for v1.11.x version)] (v1.11/README_CN.md)
- [English document(for v1.11.x version)](v1.11/README.md)
- [Chinese documentation (for v1.9.x version)] (v1.9/README_CN.md)
- [English document(for v1.9.x version)](v1.9/README.md)
- [Chinese documentation (for v1.7.x version)] (v1.7/README_CN.md)
- [English document(for v1.7.x version)](v1.7/README.md)
- [Chinese documentation (for v1.6.x version)] (v1.6/README_CN.md)
- [English document(for v1.6.x version)](v1.6/README.md)

---

- [GitHub project address] (https://github.com/cookeem/kubeadm-ha/)
- [OSChina Project Address] (https://git.oschina.net/cookeem/kubeadm-ha/)

---

## Table of Contents

- This guideline applies to the v1.14.x version of the kubernetes cluster

- [Deployment Architecture] (#Deployment Architecture)
  - [Deployment Architecture Summary] (#Deployment Architecture Summary)
  - [Host List] (#Host List)
  - [Version Information] (#Version Information)
- [Prepare before installation] (#Prepare before installation)
  - [System Update] (#System Update)
  - [Firewall Settings] (#Firewall Settings)
  - [System Parameter Settings] (#System Parameter Settings)
  - [master node mutual trust setting] (#master node mutual trust setting)
- [Installation Components] (#Installation Components)
  - [docker installation] (#docker installation)
  - [kubernetes management software installation] (#kubernetes management software installation)
  - [keepalived installation] (#keepalived installation)
- [Create Profile] (#Create Profile)
  - [Generate related configuration files] (#Generate related configuration files)
  - [Profile List] (#Profile List)
- [start load-balancer] (#start loadbalancer)
  - [start keepalived] (#start keepalived)
  - [Start nginx-lb] (#start nginxlb)
- [Initialize high availability master cluster] (#Initialize high availability master cluster)
  - [Get kubernetes image] (#Get kubernetes image)
  - [Install the first master node] (#Install the first master node)
  - [Add other master nodes to the control plane control plane] (#Add other master nodes to the control plane control plane)
- [Use nginx-lb as kubernetes cluster basic service] (#put nginxlb as kubernetes cluster basic service)
- [Add worker node] (#Add worker node)
- [Set kubectl auto-complete] (#Set kubectl auto-complete)
- [Installation Components] (#Installation Components)
  - [kubernetes-dashboard](#kubernetes-dashboard)
    - [Install kubernetes-dashboard] (#install kubernetes-dashboard)
    - [Log in kubernetes-dashboard] (#Login kubernetes-dashboard)
    - [Manage clusters with kubernetes-dashboard] (#Using kubernetes-dashboard to manage clusters)
  - [heapster](#heapster)
    - [Install heapster] (#install heapster)
    - [Verify heapster metric information collection] (#Verify heapster metric information collection)
  - [metrics-server](#metrics-server)
    - [Install metrics-server] (#Install metrics-server)
  - [prometheus](#prometheus)
    - [Install prometheus related components] (#Install prometheus related components)
    - [Use prometheus to monitor performance] (#Use prometheus to monitor performance)
    - [Use alertmanager to verify the alarm] (#Use alertmanager to verify the alarm)
  - [grafana](#grafana)
    - [Install grafana] (#install grafana)
    - [Use grafana to render prometheus performance indicators] (# use grafana to present prometheus performance indicators)
  - [istio](#istio)
    - [Installing isti] (#Installing isti)
    - [Use isti for AB test] (# use isti for AB test)
    - [Perform Service Tracking] (#Service Tracking)
    - [Perform traffic monitoring] (# for traffic monitoring)
  - [traefik](#traefik)
    - [Install traefik] (#install traefik)
    - [Use traefik as a border router] (#Use traefik as a border router)
- [Cluster Verification] (#Cluster Verification)
  - [Cluster High Availability Verification Test] (#Cluster High Availability Verification Test)
  - [nodePort test] (#nodeport test)
  - [Intra-cluster service test] (#Cluster service test)
  - [Test automatic expansion and expansion] (# test automatic expansion and contraction)
- [Certificate Expiration Update] (#Certificate Expired Update)

## Deployment Architecture

### Deployment Architecture Overview

![](images/kubernetes-ha-architecture.png)

- As shown above, kubernetes high availability clusters are deployed using the `Stacked etcd topology` pattern. Each master node (control plane node) runs `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, Each master node also runs `etcd` as the cluster running state storage. The `kube-apiserver` provides services to the worker node in the form of load balancer.

### Host list

Host Name | IP Address | Description | Component
:--- | :--- | :--- | :---
Demo-vip.local | 172.20.10.9 | keepalived Floating IP | None
Demo-01.local ~ demo-03.local | 172.20.10.10 ~ 12 | master node * 3 | keepalived, nginx, kubelet, kube-apiserver, kube-scheduler, kube-controller-manager, etcd
Demo-04.local | 172.20.10.13 | worker node * 1 | kubelet

### Version Information

- System version information

```bash
# Linux Release Information
$ cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)

# Linux kernel
$ uname -a
Linux demo-01.local 5.0.4-1.el7.elrepo.x86_64 #1 SMP Sat Mar 23 18:00:44 EDT 2019 x86_64 x86_64 x86_64 GNU/Linux

# kubernetes version
$ kubelet --version
Kubernetes v1.14.0

# docker-ce version information
$ docker version
Client:
 Version: 18.09.3
 API version: 1.39
 Go version: go1.10.8
 Git commit: 774a1f4
 Built: Thu Feb 28 06:33:21 2019
 OS/Arch: linux/amd64
 Experimental: false

Server: Docker Engine - Community
 Engine:
  Version: 18.09.3
  API version: 1.39 (minimum version 1.12)
  Go version: go1.10.8
  Git commit: 774a1f4
  Built: Thu Feb 28 06:02:24 2019
  OS/Arch: linux/amd64
  Experimental: false

# docker-compose version
$ docker-compose version
Docker-compose version 1.18.0, build 8dd22a9
Docker-py version: 2.6.1
CPython version: 3.4.9
OpenSSL version: OpenSSL 1.0.2k-fips 26 Jan 2017
```

- Component version information

Component | Version | Notes
:--- | :--- | :---
Calico | v3.6.0 | Network Components
Kubernetes-dashboard | v1.10.1 | kubernetes management UI
Heapster | v1.5.4 | Performance Acquisition Component (used by kubernetes-dashboard)
Metrics-server | v0.3.1 | Performance Acquisition Component
Prometheus | v2.8.0 | Performance Monitoring Component
Alertmanager | v0.16.1 | Alarm Management Component
Node-exporter | v0.17.0 | Node Performance Acquisition Component
Kube-state-metrics | v1.5.0 | kubernetes cluster state acquisition component
Grafana | v6.0.2 | Performance Rendering Component
Istio | v1.1.1 | Service Grid
Traefik | v1.7.9 | ingress boundary routing component

## Preparation before installation

### system update

- Update operating system kernel

```bash
# Setting elrepo, linux kernel update yum source
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
$ rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

# View the latest operating system kernel version, here the version is 5.0.4-1.el7.elrepo
$ yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
Loaded plugin: fastestmirror
Loading mirror speeds from cached hostfile
 * elrepo-kernel: mirror-hk.koddos.net
Installable package
Kernel-lt.x86_64 4.4.177-1.el7.elrepo elrepo-kernel
Kernel-lt-devel.x86_64 4.4.177-1.el7.elrepo elrepo-kernel
Kernel-lt-doc.noarch 4.4.177-1.el7.elrepo elrepo-kernel
Kernel-lt-headers.x86_64 4.4.177-1.el7.elrepo elrepo-kernel
Kernel-lt-tools.x86_64 4.4.177-1.el7.elrepo elrepo-kernel
Kernel-lt-tools-libs.x86_64 4.4.177-1.el7.elrepo elrepo-kernel
Kernel-lt-tools-libs-devel.x86_64 4.4.177-1.el7.elrepo elrepo-kernel
Kernel-ml-devel.x86_64 5.0.4-1.el7.elrepo elrepo-kernel
Kernel-ml-doc.noarch 5.0.4-1.el7.elrepo elrepo-kernel
Kernel-ml-headers.x86_64 5.0.4-1.el7.elrepo elrepo-kernel
Kernel-ml-tools.x86_64 5.0.4-1.el7.elrepo elrepo-kernel
Kernel-ml-tools-libs.x86_64 5.0.4-1.el7.elrepo elrepo-kernel
Kernel-ml-tools-libs-devel.x86_64 5.0.4-1.el7.elrepo elrepo-kernel
Perf.x86_64 5.0.4-1.el7.elrepo elrepo-kernel
Python-perf.x86_64 5.0.4-1.el7.elrepo elrepo-kernel

# Install the new version of the kernel kernel-ml-5.0.4-1.el7.elrepo
$ yum --disablerepo="*" --enablerepo="elrepo-kernel" install -y kernel-ml-5.0.4-1.el7.elrepo

# Modify the startup options of /etc/default/grub
$ vi /etc/default/grub
GRUB_DEFAULT=0

# Create a grub startup configuration file
$ grub2-mkconfig -o /boot/grub2/grub.cfg

# Restart the system
$ reboot

# Verify system version, has been updated to 5.0.4-1.el7.elrepo.x86_64
$ uname -a
Linux demo-01.local 5.0.4-1.el7.elrepo.x86_64 #1 SMP Sat Mar 23 18:00:44 EDT 2019 x86_64 x86_64 x86_64 GNU/Linux
```

- Update system package version

```bash
# Update system package version
$ yum update -y
```

### Firewall settings

- All nodes open the firewalld firewall

```sh
# Restart the firewall
$ systemctl enable firewalld
$ systemctl restart firewalld
$ systemctl status firewalld
```

- The master node needs an open port

Protocol | Direction | Port | Description
:--- | :--- | :--- | :---
TCP | Inbound | 16443* | Load balancer Kubernetes API server port
TCP | Inbound | 6443* | Kubernetes API server
TCP | Inbound | 4001 | etcd listen client port
TCP | Inbound | 2379-2380 | etcd server client API
TCP | Inbound | 10250 | Kubelet API
TCP | Inbound | 10251 | kube-scheduler
TCP | Inbound | 10252 | kube-controller-manager
TCP | Inbound | 30000-32767 | NodePort Services

- All master nodes set firewall policies

```bash
#open port policy
$ firewall-cmd --zone=public --add-port=16443/tcp --permanent
$ firewall-cmd --zone=public --add-port=6443/tcp --permanent
$ firewall-cmd --zone=public --add-port=4001/tcp --permanent
$ firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent
$ firewall-cmd --zone=public --add-port=10250/tcp --permanent
$ firewall-cmd --zone=public --add-port=10251/tcp --permanent
$ firewall-cmd --zone=public --add-port=10252/tcp --permanent
$ firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
$ firewall-cmd --reload

# View port open status
$ firewall-cmd --list-all --zone=public
Public (active)
  Target: default
  Icmp-block-inversion: no
  Interfaces: ens2f1 ens1f0 nm-bond
  Sources:
  Services: ssh dhcpv6-client
  Ports: 4001/tcp 6443/tcp 2379-2380/tcp 10250/tcp 10251/tcp 10252/tcp 30000-32767/tcp
  Protocols:
  Masquerade: no
  Forward-ports:
  Source-ports:
  Icmp-blocks:
  Rich rules:
```

- The worker node needs an open port

Protocol | Direction | Port | Description
:--- | :--- | :--- | :---
TCP | Inbound | 10250 | Kubelet API
TCP | Inbound | 30000-32767 | NodePort Services

- All worker nodes set firewall policies

```sh
#open port policy
$ firewall-cmd --zone=public --add-port=10250/tcp --permanent
$ firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
$ firewall-cmd --reload

# View port open status
$ firewall-cmd --list-all --zone=public
Public (active)
  Target: default
  Icmp-block-inversion: no
  Interfaces: ens2f1 ens1f0 nm-bond
  Sources:
  Services: ssh dhcpv6-client
  Ports: 10250/tcp 30000-32767/tcp
  Protocols:
  Masquerade: no
  Forward-ports:
  Source-ports:
  Icmp-blocks:
  Rich rules:
```

- All nodes must clear the icmp forbidden policy on iptables, otherwise the cluster network detection will be abnormal.

```bash
# To prevent the firewalld service from restarting iptables is reset, the policy update is regularly updated in the crontab
$ crontab -e
* * * * * /usr/sbin/iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited
```

- Set the `/etc/hosts` hostname for all nodes, please configure according to the actual situation.

```bash
$ cat /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
172.20.10.9 demo-vip.local
172.20.10.10 demo-01.local
172.20.10.11 demo-02.local
172.20.10.12 demo-03.local
172.20.10.13 demo-04.local
```

### System parameter setting

- Set SELINUX to permissive mode on all nodes

```sh
# Change setting
$ vi /etc/selinux/config
SELINUX=permissive

$ setenforce 0
```

- Set iptables parameters on all nodes

```sh
# All nodes configured ip forwarding
$ cat <<EOF > /etc/sysctl.d/k8s.conf
Net.bridge.bridge-nf-call-ip6tables = 1
Net.bridge.bridge-nf-call-iptables = 1
EOF

# Let the configuration take effect
$ sysctl --system
```

- Disable swap on all nodes

```sh
$ swapoff -a

# Disable the swap project in fstab
$ vi /etc/fstab
#/dev/mapper/centos-swap swap swap defaults 0 0

# Confirm that swap has been disabled
$ cat /proc/swaps
Filename Type Size Used Priority

# Restart the host
$ reboot
```

- Note that installing the kubernetes cluster requires at least 2 CPUs per node.

### master node mutual trust setting

- Mutual password-free login settings on all master nodes

```bash
# Create a local public key on all master nodes
$ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa

# Append id_rsa.pub (public key) to the authorization key file on demo-01.local
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# Copy the authorization key file to demo-02.local on demo-01.local
$ scp ~/.ssh/authorized_keys root@demo-02.local:~/.ssh/

# Append id_rsa.pub (public key) to the authorization key file on demo-02.local
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# Copy the authorization key file to demo-03.local on demo-02.local
$ scp ~/.ssh/authorized_keys root@demo-03.local:~/.ssh/

# Append id_rsa.pub (public key) to the authorization key file on demo-03.local
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# Copy the authorization key file to demo-01.local and demo-02.local on demo-03.local
$ scp ~/.ssh/authorized_keys root@demo-01.local:~/.ssh/
$ scp ~/.ssh/authorized_keys root@demo-02.local:~/.ssh/

# Execute ssh on all master nodes to check whether passwords are trusted for mutual login.
$ ssh demo-01.local hostname && \
Ssh demo-02.local hostname && \
Ssh demo-03.local hostname
Demo-01.local
Demo-02.local
Demo-03.local
```

## Installing components

### docker installation

- Set the docker-ce installation yum source

```bash
# Install yum management tool
$ yum install -y yum-utils

# Add Alibaba Cloud's yum source
$ yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

- Install docker-ce

```bash
# View the available docker-ce version, the latest version is 18.09.3-3.el7
$ yum list docker-ce --showduplicates
Loaded plugin: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * elrepo: mirror-hk.koddos.net
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Installed package
Docker-ce.x86_64 3:18.09.3-3.el7 @docker-ce-stable
Installable package
Docker-ce.x86_64 17.03.0.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 17.03.1.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 17.03.2.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 17.03.3.ce-1.el7 docker-ce-stable
Docker-ce.x86_64 17.06.0.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 17.06.1.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 17.06.2.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 17.09.0.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 17.09.1.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 17.12.0.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 17.12.1.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 18.03.0.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 18.03.1.ce-1.el7.centos docker-ce-stable
Docker-ce.x86_64 18.06.0.ce-3.el7 docker-ce-stable
Docker-ce.x86_64 18.06.1.ce-3.el7 docker-ce-stable
Docker-ce.x86_64 18.06.2.ce-3.el7 docker-ce-stable
Docker-ce.x86_64 18.06.3.ce-3.el7 docker-ce-stable
Docker-ce.x86_64 3:18.09.0-3.el7 docker-ce-stable
Docker-ce.x86_64 3:18.09.1-3.el7 docker-ce-stable
Docker-ce.x86_64 3:18.09.2-3.el7 docker-ce-stable
Docker-ce.x86_64 3:18.09.3-3.el7 docker-ce-stable

# Install docker-ce
$ yum install -y 3:docker-ce-18.09.3-3.el7.x86_64

#Start docker service
$ systemctl enable docker && systemctl start docker

# View docker version
$ docker version
Client:
 Version: 18.09.3
 API version: 1.39
 Go version: go1.10.8
 Git commit: 774a1f4
 Built: Thu Feb 28 06:33:21 2019
 OS/Arch: linux/amd64
 Experimental: false

Server: Docker Engine - Community
 Engine:
  Version: 18.09.3
  API version: 1.39 (minimum version 1.12)
  Go version: go1.10.8
  Git commit: 774a1f4
  Built: Thu Feb 28 06:02:24 2019
  OS/Arch: linux/amd64
  Experimental: false

# Install docker-compose, docker-compose for temporary startup of nginx-lb during installation
$ yum install docker-compose
```

### kubernetes management software installation

- Set kubernetes to install yum source

```bash
# Configure kubernetes software yum source
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
Name=Kubernetes
Baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
Enabled=1
Gpgcheck=1
Repo_gpgcheck=1
Gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

- Install kubernetes

```bash
# View the kubernetes version on yum, the latest version here is kubelet-1.14.0-0.x86_64
$ yum search kubelet --showduplicates
...
Kubelet-1.12.0-0.x86_64 : Container cluster management
Kubelet-1.12.1-0.x86_64 : Container cluster management
Kubelet-1.12.2-0.x86_64 : Container cluster management
Kubelet-1.12.3-0.x86_64 : Container cluster management
Kubelet-1.12.4-0.x86_64 : Container cluster management
Kubelet-1.12.5-0.x86_64 : Container cluster management
Kubelet-1.12.6-0.x86_64 : Container cluster management
Kubelet-1.12.7-0.x86_64 : Container cluster management
Kubelet-1.13.0-0.x86_64 : Container cluster management
Kubelet-1.13.1-0.x86_64 : Container cluster management
Kubelet-1.13.2-0.x86_64 : Container cluster management
Kubelet-1.13.3-0.x86_64 : Container cluster management
Kubelet-1.13.4-0.x86_64 : Container cluster management
Kubelet-1.13.5-0.x86_64 : Container cluster management
Kubelet-1.14.0-0.x86_64 : Container cluster management
Kubelet-1.14.0-0.x86_64 : Container cluster management
...

# Install kubernetes component
$ yum install -y kubeadm-1.14.0-0.x86_64 kubelet-1.14.0-0.x86_64 kubectl-1.14.0-0.x86_64
```

### keepalived installation

- Install keepalived

```bash
# Install keepalived
$ yum install -y keepalived
```

- Start the keepalived service

```bash
#Start keepalived service
$ systemctl enable keepalived && systemctl start keepalived
```

## Creating a configuration file

- Pull the [https://github.com/cookeem/kubeadm-ha](https://github.com/cookeem/kubeadm-ha) code and put the kubeadm-ha code on the first master node `demo- 01.local` /root directory

```bash
$ git clone https://github.com/cookeem/kubeadm-ha
```

- The core configuration file is as follows

```bash
.
├── create-config.sh #Automatically generate scripts for related configuration files
├── kubeadm-config.yaml.tpl # kubeadm initialization profile template
├── calico
│ └── calico.yaml.tpl # calico network component profile template
├── keepalived
│ ├── check_apiserver.sh # keepalived automatic detection script
│ └── keepalived.conf.tpl # keepalived profile template
└── nginx-lb
 ├── docker-compose.yaml # Launch the nginx-lb configuration file using docker-compose
 ├── nginx-lb.conf.tpl # nginx-lb configuration file
 └── nginx-lb.yaml # Use kubelet to host nginx-lb configuration file
```

### Generate related configuration files

- Configure the `create-config.sh` script. Be sure to configure the following according to the cluster.

```bash
# master keepalived virtual ip address
Export K8SHA_VIP=172.20.10.9

# master01 ip address
Export K8SHA_IP1=172.20.10.10

# master02 ip address
Export K8SHA_IP2=172.20.10.11

# master03 ip address
Export K8SHA_IP3=172.20.10.12

# master keepalived virtual ip hostname
Export K8SHA_VHOST=demo-vip.local

# master01 hostname
Export K8SHA_HOST1=demo-01.local

# master02 hostname
Export K8SHA_HOST2=demo-02.local

# master03 hostname
Export K8SHA_HOST3=demo-03.local

# master01 network interface name
Export K8SHA_NETINF1=enp0s3

# master02 network interface name
Export K8SHA_NETINF2=enp0s3

# master03 network interface name
Export K8SHA_NETINF3=enp0s3

# keepalived auth_pass config
Export K8SHA_KEEPALIVED_AUTH=412f7dc3bfed32194d1600c483e10ad1d

# calico reachable ip address
Export K8SHA_CALICO_REACHABLE_IP=172.20.10.1

# kubernetes CIDR pod subnet
Export K8SHA_CIDR=192.168.0.0
```

- Execute the configuration script to generate the configuration file

```bash
# Execute script to automatically create a configuration file
$ ./create-config.sh
Create kubeadm-config.yaml files success. kubeadm-config.yaml
Create keepalived files success. config/demo-01.local/keepalived/
Create keepalived files success. config/demo-02.local/keepalived/
Create keepalived files success. config/demo-03.local/keepalived/
Create nginx-lb files success. config/demo-01.local/nginx-lb/
Create nginx-lb files success. config/demo-02.local/nginx-lb/
Create nginx-lb files success. config/demo-03.local/nginx-lb/
Create calico.yaml file success. calico/calico.yaml
Docker-compose.yaml 100% 213 162.8KB/s 00:00
Nginx-lb.yaml 100% 680 461.9KB/s 00:00
Nginx-lb.conf 100% 1033 1.3MB/s 00:00
Docker-compose.yaml 100% 213 113.9KB/s 00:00
Nginx-lb.yaml 100% 680 439.6KB/s 00:00
Nginx-lb.conf 100% 1033 854.1KB/s 00:00
Docker-compose.yaml 100% 213 86.3KB/s 00:00
Nginx-lb.yaml 100% 680 106.7KB/s 00:00
Nginx-lb.conf 100% 1033 542.8KB/s 00:00
Check_apiserver.sh 100% 488 431.4KB/s 00:00
Keepalived.conf 100% 558 692.5KB/s 00:00
Check_apiserver.sh 100% 488 360.3KB/s 00:00
Keepalived.conf 100% 558 349.0KB/s 00:00
Check_apiserver.sh 100% 488 371.4KB/s 00:00
Keepalived.conf 100% 558 382.4KB/s 00:00
```

### Configuration file list

- After executing the `create-config.sh` script, the following configuration files are automatically generated:

  - kubeadm-config.yaml: kubeadm initialization configuration file, located in the `./` root directory of the kubeadm-ha code

  - keepalived: keepalived configuration file, located in the `/etc/keepalived` directory of each master node

  - nginx-lb: nginx-lb load balancing configuration file, located in the `/root/nginx-lb` directory of each master node

  - calico.yaml: calico network component deployment file, located in the `./calico` directory of the kubeadm-ha code

## Start load-balancer

- The kubeadm installation method of kubernetes v1.14.0 needs to start the load-balancer first. Here we use keepalived as the floating ip and nginx as the load balancing.

### Start keepalived

- After executing the `create-config.sh` script, the keepalived configuration file is automatically copied to the `/etc/keepalived` directory of each master node.

- Restart the keepalived service on all master nodes

```bash
# Restart keepalived service
$ systemctl restart keepalived

# View the status of the keepalived service. The following information indicates that keepalived has started normally.
$ systemctl status keepalived
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since five 2019-03-29 02:25:32 EDT; 19min ago
  Process: 3551 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 3577 (keepalived)
    Tasks: 3
   Memory: 448.5M
   CGroup: /system.slice/keepalived.service
           ├─3577 /usr/sbin/keepalived -D
           ├─3578 /usr/sbin/keepalived -D
           └─3579 /usr/sbin/keepalived -D

March 29 02:25:54 demo-01.local Keepalived_vrrp[3579]: /etc/keepalived/check_apiserver....5
March 29 02:25:58 demo-01.local Keepalived_vrrp[3579]: Sending gratuitous ARP on enp0s3...9
March 29 02:25:58 demo-01.local Keepalived_vrrp[3579]: VRRP_Instance(VI_1) Sending/queu...9
March 29 02:25:58 demo-01.local Keepalived_vrrp[3579]: Sending gratuitous ARP on enp0s3...9
March 29 02:25:58 demo-01.local Keepalived_vrrp[3579]: Sending gratuitous ARP on enp0s3...9
March 29 02:25:58 demo-01.local Keepalived_vrrp[3579]: Sending gratuitous ARP on enp0s3...9
March 29 02:25:58 demo-01.local Keepalived_vrrp[3579]: Sending gratuitous ARP on enp0s3...9
March 29 02:25:59 demo-01.local Keepalived_vrrp[3579]: /etc/keepalived/check_apiserver....5
March 29 02:26:04 demo-01.local Keepalived_vrrp[3579]: VRRP_Script(check_apiserver) suc...d
March 29 02:26:09 demo-01.local Keepalived_vrrp[3579]: VRRP_Instance(VI_1) Changing eff...2
Hint: Some lines were ellipsized, use -l to show in full.

# Verify that the keepiped vip takes effect and the demo-vip.local is in effect.
$ ping demo-vip.local
PING demo-vip.local (172.20.10.9) 56(84) bytes of data.
64 bytes from demo-vip.local (172.20.10.9): icmp_seq=1 ttl=64 time=0.121 ms
64 bytes from demo-vip.local (172.20.10.9): icmp_seq=2 ttl=64 time=0.081 ms
64 bytes from demo-vip.local (172.20.10.9): icmp_seq=3 ttl=64 time=0.053 ms
```

### Launch nginx-lb

- After executing the `create-config.sh` script, the nginx-lb configuration file is automatically copied to the `/root/nginx-lb` directory of each master node.

- Launch nginx-lb load balancer in docker-compose mode on all master nodes

```bash
# Enter /root/nginx-lb/
$ cd /root/nginx-lb/

# Launch nginx-lb using docker-compose
$ docker-compose up -d
Creating nginx-lb ... done

# Check the startup status of nginx-lb, if the normal startup indicates that the service is normal.
$ docker-compose ps
  Name Command State Ports
-------------------------------------------------- ------------------------
Nginx-lb nginx -g daemon off; Up 0.0.0.0:16443->16443/tcp, 80/tcp
```

## Initializing a highly available master cluster

### Get kubernetes image

- It is recommended to pre-pulse the required image of kubernetes

```bash
# Get a list of mirrors required for kubernetes
$ kubeadm --kubernetes-version=v1.14.0 config images list
K8s.gcr.io/kube-apiserver: v1.14.0
K8s.gcr.io/kube-controller-manager: v1.14.0
K8s.gcr.io/kube-scheduler: v1.14.0
K8s.gcr.io/kube-proxy: v1.14.0
K8s.gcr.io/pause:3.1
K8s.gcr.io/etcd: 3.3.10
K8s.gcr.io/coredns: 1.3.1

# Pull the image required by kubernetes
$ kubeadm --kubernetes-version=v1.14.0 config images pull
```

## Initializing a highly available master cluster

### Get kubernetes image

- It is recommended to pre-pulse the required image of kubernetes

```bash
# Get a list of mirrors required for kubernetes
$ kubeadm --kubernetes-version=v1.14.0 config images list
K8s.gcr.io/kube-apiserver: v1.14.0
K8s.gcr.io/kube-controller-manager: v1.14.0
K8s.gcr.io/kube-scheduler: v1.14.0
K8s.gcr.io/kube-proxy: v1.14.0
K8s.gcr.io/pause:3.1
K8s.gcr.io/etcd: 3.3.10
K8s.gcr.io/coredns: 1.3.1

# Pull the image required by kubernetes
$ kubeadm --kubernetes-version=v1.14.0 config images pull
```

### Install the first master node

- Start the kubernetes service as master (control plane) on the first master node `demo-01.local`

```bash
# Execute the following command on demo-01.local to initialize the first master node and wait a few minutes.
# Very important, be sure to save the following contents in the execution result. There are two commands in the output result to join the node to the cluster as the master or worker.
$ kubeadm init --config=/root/kubeadm-config.yaml --experimental-upload-certs
...
You can now join any number of the control-plane node running the following command on each as root:

  Kubeadm join 172.20.10.9:16443 --token m8k0m4.eheru9acqftbre89 \
    --discovery-token-ca-cert-hash sha256:e97e9db0ca6839cae2989571346b6142f7e928861728d5067a979668aaf46954 \
    --experimental-control-plane --certificate-key b1788f02d442c623d28f5281bb566bf4fcd9e739c45f127a95ea07b558538244

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --experimental-upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

Kubeadm join 172.20.10.9:16443 --token m8k0m4.eheru9acqftbre89 \
    --discovery-token-ca-cert-hash sha256:e97e9db0ca6839cae2989571346b6142f7e928861728d5067a979668aaf46954
```

- Set the configuration file of kubectl, you can use kubectl to access the cluster after setting.

```bash
# Set the KUBECONFIG environment variable in the ~/.bashrc file
$ cat << EOF >> ~/.bashrc
Export KUBECONFIG=/etc/kubernetes/admin.conf
EOF

# kuBECONFIG can be used to access the cluster using kubectl
$ source ~/.bashrc
```

- Install calico network components, otherwise coredns will not start properly

```bash
# calico network components need to pull the following image first
$ docker pull calico/cni:v3.6.0
$ docker pull calico/node: v3.6.0
$ docker pull calico/kube-controllers: v3.6.0

# Install the calico component on demo-01.local
$ kubectl apply -f calico/calico.yaml
Configmap/calico-config created
Customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
Customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
Clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
Clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
Clusterrole.rbac.authorization.k8s.io/calico-node created
Clusterrolebinding.rbac.authorization.k8s.io/calico-node created
Daemonset.extensions/calico-node created
Serviceaccount/calico-node created
Deployment.extensions/calico-kube-controllers created
Serviceaccount/calico-kube-controllers created

# Check the pod status after the calico installation, the coredns status returns to normal.
$ kubectl get pods --all-namespaces
NAMESPACE NAME READY STATUS RESTARTS AGE
Kube-system calico-kube-controllers-7bfdd87774-gh4rf 1/1 Running 0 24s
Kube-system calico-node-4kcrh 1/1 Running 0 24s
Kube-system coredns-fb8b8dccf-7fp98 1/1 Running 0 4m42s
Kube-system coredns-fb8b8dccf-l8xzz 1/1 Running 0 4m42s
Kube-system etcd-demo-01.local 1/1 Running 0 4m15s
Kube-system kube-apiserver-demo-01.local 1/1 Running 0 3m59s
Kube-system kube-controller-manager-demo-01.local 1/1 Running 0 3m58s
Kube-system kube-proxy-qf9zp 1/1 Running 0 4m42s
Kube-system kube-scheduler-demo-01.local 1/1 Running 0 4m17s

# View node status, demo-01.local status is normal
$ kubectl get nodes
NAME STATUS ROLES AGE VERSION
Demo-01.local Ready master 5m3s v1.14.0
```

### Add other master nodes to the control plane control plane

- After the first master node uses kubeadm to initialize the first kubernetes master node, it will output two commands, the first of which is to add the node to the cluster as master.

- Execute the following commands on the second and third master nodes `demo-02.local` and `demo-03.local` respectively, and add `demo-02.local` and `demo-03.local` as master. To the cluster.

```bash
# Execute the following command on the other master nodes demo-02.local and demo-03.local to add the node to the control plane.
$ kubeadm join 172.20.10.9:16443 --token m8k0m4.eheru9acqftbre89 \
  --discovery-token-ca-cert-hash sha256:e97e9db0ca6839cae2989571346b6142f7e928861728d5067a979668aaf46954 \
  --experimental-control-plane --certificate-key b1788f02d442c623d28f5281bb566bf4fcd9e739c45f127a95ea07b558538244
```

- Set the configuration file of kubectl on `demo-02.local` and `demo-03.local`, and then use kubectl to access the cluster after setting.

```bash
# Set the KUBECONFIG environment variable in the ~/.bashrc file
$ cat << EOF >> ~/.bashrc
Export KUBECONFIG=/etc/kubernetes/admin.conf
EOF

# kuBECONFIG can be used to access the cluster using kubectl
$ source ~/.bashrc
```

- Check the cluster status

```bash
# View node status
$ kubectl get nodes
NAME STATUS ROLES AGE VERSION
Demo-01.local Ready master 10m v1.14.0
Demo-02.local Ready master 3m10s v1.14.0
Demo-03.local Ready master 110s v1.14.0

# View pod status
$ kubectl get pods -n kube-system -o wide
NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES
Calico-kube-controllers-7bfdd87774-gh4rf 1/1 Running 0 5m22s 192.168.8.65 demo-01.local <none> <none>
Calico-node-4kcrh 1/1 Running 0 5m22s 172.20.10.10 demo-01.local <none> <none>
Calico-node-4ljlm 1/1 Running 0 3m2s 172.20.10.11 demo-02.local <none> <none>
Calico-node-wh5rs 1/1 Running 0 102s 172.20.10.12 demo-03.local <none> <none>
Coredns-fb8b8dccf-7fp98 1/1 Running 0 9m40s 192.168.8.67 demo-01.local <none> <none>
Coredns-fb8b8dccf-l8xzz 1/1 Running 0 9m40s 192.168.8.66 demo-01.local <none> <none>
Etcd-demo-01.local 1/1 Running 0 9m13s 172.20.10.10 demo-01.local <none> <none>
Etcd-demo-02.local 1/1 Running 0 3m1s 172.20.10.11 demo-02.local <none> <none>
Etcd-demo-03.local 1/1 Running 0 100s 172.20.10.12 demo-03.local <none> <none>
Kube-apiserver-demo-01.local 1/1 Running 0 8m57s 172.20.10.10 demo-01.local <none> <none>
Kube-apiserver-demo-02.local 1/1 Running 0 3m2s 172.20.10.11 demo-02.local <none> <none>
Kube-apiserver-demo-03.local 1/1 Running 0 101s 172.20.10.12 demo-03.local <none> <none>
Kube-controller-manager-demo-01.local 1/1 Running 0 8m56s 172.20.10.10 demo-01.local <none> <none>
Kube-controller-manager-demo-02.local 1/1 Running 0 3m2s 172.20.10.11 demo-02.local <none> <none>
Kube-controller-manager-demo-03.local 1/1 Running 0 101s 172.20.10.12 demo-03.local <none> <none>
Kube-proxy-mt6z6 1/1 Running 0 102s 172.20.10.12 demo-03.local <none> <none>
Kube-proxy-qf9zp 1/1 Running 0 9m40s 172.20.10.10 demo-01.local <none> <none>
Kube-proxy-sfhrs 1/1 Running 0 3m2s 172.20.10.11 demo-02.local <none> <none>
Kube-scheduler-demo-01.local 1/1 Running 0 9m15s 172.20.10.10 demo-01.local <none> <none>
Kube-scheduler-demo-02.local 1/1 Running 0 3m2s 172.20.10.11 demo-02.local <none> <none>
Kube-scheduler-demo-03.local 1/1 Running 0 101s 172.20.10.12 demo-03.local <none> <none>
```

## Using nginx-lb as the basic service of kubernetes cluster

- High-availability kubernetes cluster has been configured, but nginx-lb is started using docker-compose. Since the health check and auto-restart function of the kubernetes cluster is not available, nginx-lb is recommended as a core component of the highly available kubernetes cluster as well as in the kubernetes cluster. A pod to manage.

- In the /etc/kubernetes/manifests/ directory, it is the core deployment file directly managed by kubelet. Now we stop the nginx-lb that was originally started with docker-compose, and the kubelet directly manages the nginx-lb service.

```bash
# All nodes suspend kubelet
$ systemctl stop kubelet

# All nodes stop and delete the nginx-lb container launched using docker-compose mode
$ docker stop nginx-lb && docker rm nginx-lb

# Execute the following command on the demo-01.local node to copy the nginx-lb configuration to the /etc/kubernetes/manifests/ directory of all master nodes.
$ export K8SHA_HOST1=demo-01.local
$ export K8SHA_HOST2=demo-02.local
$ export K8SHA_HOST3=demo-03.local
$ scp /root/nginx-lb/nginx-lb.conf root@${K8SHA_HOST1}:/etc/kubernetes/
$ scp /root/nginx-lb/nginx-lb.conf root@${K8SHA_HOST2}:/etc/kubernetes/
$ scp /root/nginx-lb/nginx-lb.conf root@${K8SHA_HOST3}:/etc/kubernetes/
$ scp /root/nginx-lb/nginx-lb.yaml root

## Add worker node

- After the first master node uses kubeadm to initialize the first kubernetes master node, it will output two commands. The second command is to add the node to the cluster as a worker.

- Execute the following command on the worker node `demo-04.local` and add `demo-04.local` to the cluster as a worker.

```bash
# Run the following command on the demo-04.local node to add the node to the cluster as a worker.
$ kubeadm join 172.20.10.9:16443 --token m8k0m4.eheru9acqftbre89 \
    --discovery-token-ca-cert-hash sha256:e97e9db0ca6839cae2989571346b6142f7e928861728d5067a979668aaf46954

# View cluster node status
$ kubectl get nodes
NAME STATUS ROLES AGE VERSION
Demo-01.local Ready master 99m v1.14.0
Demo-02.local Ready master 92m v1.14.0
Demo-03.local Ready master 90m v1.14.0
Demo-04.local Ready <none> 14s v1.14.0
```

## Setting kubectl to complete automatically

```bash
# Install bash-completion
$ yum install -y bash-completion

# Set kubectl to complete automatically
$ source <(kubectl completion bash)
$ kubectl completion bash > ~/.kube/completion.bash.inc
  Printf "
  # Kubectl shell completion
  Source '$HOME/.kube/completion.bash.inc'
  " >> $HOME/.bash_profile
  Source $HOME/.bash_profile
```

## Installing components

### kubernetes-dashboard

#### Installing kubernetes-dashboard

#### Login kubernetes-dashboard

#### Using kubernetes-dashboard to manage clusters

### heapster

#### Installing heapster

#### Verify heapster metrics information collection

### metrics-server

#### Installing metrics-server

### prometheus

#### Installing prometheus related components

#### Monitoring performance with prometheus

#### Verifying alarms with alertmanager

### grafana

#### Installing grafana

#### Using grafana to present prometheus performance indicators

### istio

#### Installingistio

#### Using istio for AB testing

#### Conducting service tracking

#### Performing traffic monitoring

### traefik

#### Installing traefik

#### Using traefik as a border router

## Cluster verification

### Cluster High Availability Verification Test

### nodePort test

### Intra-cluster service testing

### Testing automatic expansion and contraction

## Certificate expiration update

```bash

# 允许master部署应用
$ kubectl taint nodes --all node-role.kubernetes.io/master-

# 安装kubernetes-dashboard
$ kubectl label nodes demo-01.local app=kube-system
$ kubectl label nodes demo-02.local app=kube-system
$ kubectl label nodes demo-03.local app=kube-system

$ kubectl apply -f /root/kubeadm-ha/kubernetes-dashboard/kubernetes-dashboard.yaml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created

# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-96m52
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: fde9bd21-5108-11e9-8d69-08002712c9f2

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTk2bTUyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJmZGU5YmQyMS01MTA4LTExZTktOGQ2OS0wODAwMjcxMmM5ZjIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.EkJE7RI89PdrDQrb8NMBGc7oOIlB3P2KEUUcKJO6jbun7YNW8ho_6vJUEgUuIgWoFqjb_jtMcuYHzw9Lo_Q8HsNih5a3GsuJcYFD7qFFOdroYf62FxAr82v8dYBmt2EzGy_yLK6SNibEeAIOFffosL7reIsVs3LcJMQTa2Q-aD9NrwXwdB3B90NZvzd1h6fjArObXKrwe5oeTLVgLFctXTV0hk9SNQxp6ptKpS4fxzMe8pzvI0Ft--FbG4vW2f0Cbd-hAAYi8eJyo65ndhQoq7-bYp2OFu6LbLnTCSPs8D10z0Wnv6o2RDA6Avgg7KT0M_zIRRiHubCJNDmwlTQk3Q

# 安装heapster

# 重建heapster-influxdb-amd64:v1.5.2镜像，修改/etc/config.toml文件
$ docker run -ti --rm --entrypoint "/bin/sh" k8s.gcr.io/heapster-influxdb-amd64:v1.5.2
sed -i "s/localhost/127.0.0.1/g" /etc/config.toml

$ docker ps | grep heapster | grep bin
a9aa804c95d7        k8s.gcr.io/heapster-influxdb-amd64:v1.5.2   "/bin/sh"                2 minutes ago       Up 2 minutes                            priceless_wilson

$ docker commit priceless_wilson k8s.gcr.io/heapster-influxdb-amd64:v1.5.2-fixed

$ kubectl apply -f /root/kubeadm-ha/heapster/
clusterrolebinding.rbac.authorization.k8s.io/heapster created
clusterrole.rbac.authorization.k8s.io/heapster created
serviceaccount/heapster created
deployment.extensions/heapster created
service/heapster created
deployment.extensions/monitoring-influxdb created
service/monitoring-influxdb created

$ kubectl top pods -n kube-system
NAME                                       CPU(cores)   MEMORY(bytes)
calico-kube-controllers-7bfdd87774-gh4rf   1m           13Mi
calico-node-4kcrh                          21m          66Mi
calico-node-4ljlm                          23m          65Mi
calico-node-wh5rs                          23m          65Mi
calico-node-wwmpf                          18m          64Mi
coredns-fb8b8dccf-7fp98                    2m           11Mi
coredns-fb8b8dccf-l8xzz                    2m           11Mi
etcd-demo-01.local                         39m          89Mi
etcd-demo-02.local                         52m          84Mi
etcd-demo-03.local                         38m          82Mi
heapster-665bbb7c6f-wd559                  1m           28Mi
kube-apiserver-demo-01.local               77m          229Mi
kube-apiserver-demo-02.local               68m          243Mi
kube-apiserver-demo-03.local               77m          219Mi
kube-controller-manager-demo-01.local      19m          51Mi
kube-controller-manager-demo-02.local      1m           16Mi
kube-controller-manager-demo-03.local      1m           14Mi
kube-proxy-9mcnp                           3m           15Mi
kube-proxy-mt6z6                           1m           14Mi
kube-proxy-qf9zp                           1m           15Mi
kube-proxy-sfhrs                           0m           14Mi
kube-scheduler-demo-01.local               11m          16Mi
kube-scheduler-demo-02.local               1m           13Mi
kube-scheduler-demo-03.local               11m          16Mi
kubernetes-dashboard-5688c4f8bd-g6fnc      0m           20Mi
monitoring-influxdb-fb5756876-xx56f        0m           21Mi
nginx-lb-demo-01.local                     31m          3Mi
nginx-lb-demo-02.local                     0m           2Mi
nginx-lb-demo-03.local                     0m           2Mi

$ kubectl taint nodes --all node-role.kubernetes.io/master-
```
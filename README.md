# !!!!!!!!  UNDER CONSTRACTION  !!!!!!!

## Main info

This playbook helps you setting up a Kubernetes Cluster on VMs or bare-metal servers.
The entire installation is performed under regular user account (but which is sudo user).
Forked from jmutai/k8s-pre-bootstrap, thanks to him.

## Supported OS

- CentOS 7

## Documentation

https://kubernetes.io/docs/setup/production-environment/

## Additional playbooks

- **net_config_copy.yml** - Copy /etc/host and /etc/resolv.conf to another servers.
- **proxy_settings_copy.yml** - Copy /etc/environment and /etc/yum.conf to another servers.  
- **prepare_os.yml** - Prepare OS.
- **prepare_os_others.yml** - Prepare OS on other servers not included in the cluster (for example, rdbms and etc.)
- **check_uniq.yml** - Verify the MAC address and product_uuid are unique for every node (https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#verify-mac-address)
- **send_public_key.yml** - Deploy the public key to remote hosts (for ssh)
- **init_cluster.yml** - Creating a single control-plane cluster with kubeadm (https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

## Preliminary preparation of the master server

- Install git and ansible on the control computer
```
sudo yum install -y epel-release 
sudo yum install -y ansible git platform-python nano
```

- Clone the Git Project to folder ~/ansible:
```
cd ~
mkdir ansible
cd ansible
git clone https://github.com/MinistrBob/k8s-pre-bootstrap.git .
cp hosts_example hosts
```

- Setting up ansible
```
ansible --version (you can see where config)
sudo nano /etc/ansible/ansible.cfg
```

such settings
```
inventory = /home/<user>/ansible/hosts
host_key_checking = False
```

- Edit file hosts (list of servers). Example in hosts_example.
```
nano hosts
```

# Deploy the public key to remote hosts (setup passwordless authentication) and visudo current user

- Generate keys
```
ssh-keygen -t rsa
```

- Edit **send_public_key.yml**, insert instead of ```<sshkey>``` line with public key from ```/home/<user>/.ssh/id_rsa.pub```. You can see key ```cat /home/<user>/.ssh/id_rsa.pub```.

- Edit **send_public_key.yml**, insert instead of ```<username>``` user name (user under whom the installation is performed). 

- Deploy key with ansible
```
ansible-playbook send_public_key.yml -b --ask-pass
```
If you get the error "missing sudo password", try this
```
ansible-playbook send_public_key.yml -b --ask-pass -kK
```


## Verify the MAC address and product_uuid are unique for every node

Playbook **check_uniq.yml** show MAC addresses and UUID. If VMs were cloned, then they may have not uniqu MAC and UUID. **You must visually verify that everything is unique**.  
```
ansible-playbook check_uniq.yml
```

## Preliminary preparation infrastructure
  
- It is desirable that all servers distinguish each other by name. To do this, either you need to have a configured DNS or prepare files ```/etc/hosts``` and ```/etc/resolv.conf``` on the master and copy them to other servers using **net_config_copy.yml**.
- WARNING: Executed only for workers. You prepare configuration files on the Master, and then using ansible they are copied to the Workers.  
- WARNING: If you use Network Manager (in the CentOS 7 by default it that) to change the DNS settings, changing file ```/etc/resolv.conf``` is not enough, you need to change the network settings, for example in ```/etc/sysconfig/network-scripts/ifcfg-ens192```, otherwise the Network Manager will overwrite file ```/etc/resolv.conf``` when the OS reboots
```
ansible-playbook net_config_copy.yml
```

- If Internet works through a proxy, then you need configure `/etc/environment`, `/etc/yum.conf`, `/etc/profile.d/http_proxy.sh` files, and run command that copy this files to another servers.
- WARNING: Executed only for workers. You prepare configuration files on the Master, and then using ansible they are copied to the Workers.  
```
ansible-playbook proxy_settings_copy.yml
```
This is example files:  
```
# nano /etc/environment


https_proxy=http://10.1.113.15:1010/
http_proxy=http://10.1.113.15:1010/
no_proxy=localhost,127.0.0.0/8,::1,10.128.0.0/16,10.147.245.11,10.147.245.12,10.147.245.13,10.147.245.14,10.147.245.15,10.147.245.16,10.147.245.17,10.147.245.18,10.147.245.19,10.147.245.20,10.147.245.21,10.147.245.22,10.147.245.23,10.147.245.24,10.147.245.25,10.147.245.26,10.147.245.27,10.147.245.28,10.147.245.29,10.147.245.30,10.147.245.31,10.147.245.32,10.147.245.33,10.147.245.34,10.147.245.35,10.147.245.36,10.147.245.37,10.147.245.38,10.147.245.39
all_proxy=socks://10.1.113.15:1010/
ftp_proxy=http://10.1.113.15:1010/
HTTP_PROXY=http://10.1.113.15:1010/
FTP_PROXY=http://10.1.113.15:1010/
ALL_PROXY=socks://10.1.113.15:1010/
NO_PROXY=localhost,127.0.0.0/8,::1,10.128.0.0/16,10.147.245.11,10.147.245.12,10.147.245.13,10.147.245.14,10.147.245.15,10.147.245.16,10.147.245.17,10.147.245.18,10.147.245.19,10.147.245.20,10.147.245.21,10.147.245.22,10.147.245.23,10.147.245.24,10.147.245.25,10.147.245.26,10.147.245.27,10.147.245.28,10.147.245.29,10.147.245.30,10.147.245.31,10.147.245.32,10.147.245.33,10.147.245.34,10.147.245.35,10.147.245.36,10.147.245.37,10.147.245.38,10.147.245.39
HTTPS_PROXY=http://10.1.113.15:1010/
```
```
# sudo chmod 755 /etc/profile.d/http_proxy.sh
# nano /etc/profile.d/http_proxy.sh


https_proxy=http://10.1.113.15:1010/
http_proxy=http://10.1.113.15:1010/
no_proxy=localhost,127.0.0.0/8,::1,10.128.0.0/16,10.147.245.11,10.147.245.12,10.147.245.13,10.147.245.14,10.147.245.15,10.147.245.16,10.147.245.17,10.147.245.18,10.147.245.19,10.147.245.20,10.147.245.21,10.147.245.22,10.147.245.23,10.147.245.24,10.147.245.25,10.147.245.26,10.147.245.27,10.147.245.28,10.147.245.29,10.147.245.30,10.147.245.31,10.147.245.32,10.147.245.33,10.147.245.34,10.147.245.35,10.147.245.36,10.147.245.37,10.147.245.38,10.147.245.39
all_proxy=socks://10.1.113.15:1010/
ftp_proxy=http://10.1.113.15:1010/
HTTP_PROXY=http://10.1.113.15:1010/
FTP_PROXY=http://10.1.113.15:1010/
ALL_PROXY=socks://10.1.113.15:1010/
NO_PROXY=localhost,127.0.0.0/8,::1,10.128.0.0/16,10.147.245.11,10.147.245.12,10.147.245.13,10.147.245.14,10.147.245.15,10.147.245.16,10.147.245.17,10.147.245.18,10.147.245.19,10.147.245.20,10.147.245.21,10.147.245.22,10.147.245.23,10.147.245.24,10.147.245.25,10.147.245.26,10.147.245.27,10.147.245.28,10.147.245.29,10.147.245.30,10.147.245.31,10.147.245.32,10.147.245.33,10.147.245.34,10.147.245.35,10.147.245.36,10.147.245.37,10.147.245.38,10.147.245.39
HTTPS_PROXY=http://10.1.113.15:1010/
```
```
# add one string at the end of file
# nano /etc/yum.conf


[main]
cachedir=/var/cache/yum/$basearch/$releasever
keepcache=0
debuglevel=2
logfile=/var/log/yum.log
exactarch=1
obsoletes=1
gpgcheck=1
plugins=1
installonly_limit=5
bugtracker_url=http://bugs.centos.org/set_project.php?project_id=23&ref=http://bugs.centos.org/bug_report_page.php?category=yum
distroverpkg=centos-release


#  This is the default, if you make this bigger yum won't see if the metadata
# is newer on the remote and so you'll "gain" the bandwidth of not having to
# download the new metadata and "pay" for it by yum not having correct
# information.
#  It is esp. important, to have correct metadata, for distributions like
# Fedora which don't keep old packages around. If you don't like this checking
# interupting your command line usage, it's much better to have something
# manually check the metadata once an hour (yum-updatesd will do this).
# metadata_expire=90m

# PUT YOUR REPOS HERE OR IN separate files named file.repo
# in /etc/yum.repos.d
proxy=http://10.1.113.15:1010
```

- All servers must have time synchronization configured (using **install_chrony.yml**). If time synchronization is already set and working, do not perform this step. If you want to use another time synchronization program, install it manually or by editing **install_chrony.yml**.  
```
ansible-playbook install_chrony.yml
```

## !!! Prepare OS !!!
- To do this! (using prepare_os.yml). Disable SELinux, Install common packages, Disable SWAP, Load required modules, Modify sysctl entries, Update OS if it need, Reboot OS (using **prepare_os.yml**).  
First execute this on the master.  
```
ansible-playbook prepare_os.yml -e target=master
```
Then execute on the workers.  
```
ansible-playbook prepare_os.yml -e target=workers
```

## Update variables in playbook file k8s-prep.yml (presented variant when firewalld is completely removed)

In file **setup_docker.yml** specific versions of packages are indicated (taken from the documentation - https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker).  
You need to check which version of the packages is currently indicated in the documentation and, if necessary, fix it.


```
nano k8s-prep.yml
---
- name: Setup Proxy
  hosts: k8s-nodes
  # remote_user: <user>
  become: yes
  become_method: sudo
  # gather_facts: no
  vars:
    k8s_cni: calico                                      # calico, flannel
    container_runtime: docker                            # docker, cri-o, containerd
    configure_firewalld: false                            # true / false
    remove_firewalld: true                               # Set to true to remove firewalld	
    # Docker registry
    setup_proxy: false                                   # Set to true to configure proxy
    proxy_server: "proxy.example.com:8080"               # Proxy server address and port
    docker_proxy_exclude: "localhost,127.0.0.1"          # Adresses to exclude from proxy
  roles:
    - kubernetes-bootstrap
```

If you are using non root remote user, then set username and enable sudo:
```
become: yes
become_method: sudo
```

To **enable proxy**, set the value of `setup_proxy` to `true` and provide proxy details in **vars**: `proxy_server` and `docker_proxy_exclude`.  
To **remove Firewalld**, set the value of `remove_firewalld` to `true` and `configure_firewalld` to `false`.  
To **install and configure Firewalld**, set the value of `remove_firewalld` to `false` and `configure_firewalld` to `true`.  

## Running Playbook with role kubernetes-bootstrap

This playbook installed all needed software on all servers without creating the cluster itself.  
```
ansible-playbook k8s-prep.yml
```

---
## Init Cluster on Master 

```
ansible-playbook init_cluster.yml
```

In folder `/root` will be created two files:
- `cluster_initialized.txt` - Cluster creation log and command for join workers.
- `pod_network_setup.txt` - Pod network installation log. 

To check the cluster you can execute (this is as example):
```
# kubectl get nodes -o wide
NAME              STATUS   ROLES    AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
rtz-ppd-mk8s-01   Ready    master   39m   v1.19.2   10.147.245.25   <none>        CentOS Linux 7 (Core)   3.10.0-1127.19.1.el7.x86_64   docker://19.3.11
# kubectl -n kube-system get pod
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-c9784d67d-98dx8   1/1     Running   0          39m
calico-node-ml9gv                         1/1     Running   0          39m
coredns-f9fd979d6-jwdnc                   1/1     Running   0          39m
coredns-f9fd979d6-wgrqh                   1/1     Running   0          39m
etcd-rtz-ppd-mk8s-01                      1/1     Running   0          39m
kube-apiserver-rtz-ppd-mk8s-01            1/1     Running   0          39m
kube-controller-manager-rtz-ppd-mk8s-01   1/1     Running   0          39m
kube-proxy-z4qvt                          1/1     Running   0          39m
kube-scheduler-rtz-ppd-mk8s-01            1/1     Running   0          39m
```

## Join workers

Join all workers servers to cluster. Copy command for join workers from `/root/cluster_initialized.txt` to `join_workers.yml`.

```
ansible-playbook join_workers.yml
```

You can check on the master server
```
kubectl get nodes
```

## Clean up (if something went wrong while creating the cluster)

See https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#tear-down

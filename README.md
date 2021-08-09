Steps to install OKD

Step 1: Set the hostname:
*************************
```
hostnamectl set-hostname master.openshift.com	# For master node
hostnamectl set-hostname node1.openshift.com
hostnamectl set-hostname node2.openshift.com
hostnamectl set-hostname infra.openshift.com
bash
```
2) change the /etc/hosts in all the nodes
******************************************
```
~]# vim /etc/hosts
167.254.204.72 		master.openshift.com		master
167.254.204.61 		node1.openshift.com		node1
167.254.204.78 		node2.openshift.com		node2
167.254.204.62		infra.openshift.com		infra
```
3)Copy the public key in all the nodes visa-versa
***************************************************
```
Copy public key  to ~/.ssh/id_rsa.pub
Copy public key  to ~/.ssh/authorized_keys 
```
4)Install NetworkManager
*********************************
```
sudo yum install NetworkManager
sudo systemctl start NetworkManager
sudo yum install -y NetworkManager wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct
```
To disable SE-Linux, this prevents csr approval issues,
```
sudo setenforce 0
```
5) Verify that you're able to ssh to all the nodes with password less.
```
yum update -y
```
6) Install the following Packages on all nodes:
**************************************************
```
yum install -y wget git  nano net-tools docker-1.13.1 bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct openssl-devel httpd-tools NetworkManager python-cryptography python-devel python-passlib java-1.8.0-openjdk-headless "@Development Tools"

systemctl start docker && systemctl enable docker && systemctl status docker
```
7) Configure Ansible Repository on master Node only.
*********************************************************
```
sudo vim /etc/yum.repos.d/ansible.repo

[ansible]
name = Ansible Repo
baseurl = https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/
enabled = 1
gpgcheck =  0

:wq! (save and exit)
```
8) Install Ansible Package and Clone Openshift-Ansible Git Repo on Master Machine :
****************************************************************************************
```
sudo yum -y install ansible-2.6* pyOpenSSL
git clone https://github.com/openshift/openshift-ansible.git
cd openshift-ansible && git fetch && git checkout release-3.11
```
9) Goto below file and edit below mentioned ones
****************************************************
```
sudo vim /etc/ssh/sshd_config

PubkeyAuthentication yes
PasswordAuthentication yes

:wq!

systemctl restart sshd
```
10) Now Create Your Own Inventory file for Ansible as following on master Node:
https://docs.okd.io/latest/install/example_inventories.html#install-config-example-inventories (it can be found here)
************************************************************************************************************************
```
[root@master ~]# vim ~/inventory.ini

[OSEv3:children]
masters
nodes
etcd

[masters]
master.openshift.com

[etcd]
master.openshift.com

[nodes]
master.openshift.com openshift_ip=167.254.204.55 openshift_schedulable=true openshift_node_group_name='node-config-master'
node1.openshift.com openshift_ip=167.254.204.52 openshift_schedulable=true openshift_node_group_name='node-config-compute'
node2.openshift.com openshift_ip=167.254.204.64 openshift_schedulable=true openshift_node_group_name='node-config-compute'
infra.openshift.com openshift_ip=167.254.204.80 openshift_schedulable=true openshift_node_group_name='node-config-infra'

[OSEv3:vars]
debug_level=4
ansible_ssh_user=centos
ansible_become=true
openshift_enable_service_catalog=true
ansible_service_broker_install=true

openshift_node_groups=[{'name': 'node-config-master', 'labels': ['node-role.kubernetes.io/master=true']}, {'name': 'node-config-infra', 'labels': ['node-role.kubernetes.io/infra=true']}, {'name': 'node-config-compute', 'labels': ['node-role.kubernetes.io/compute=true']}]

containerized=false
os_sdn_network_plugin_name='redhat/openshift-ovs-multitenant'
openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability

deployment_type=origin
openshift_deployment_type=origin

openshift_release=v3.11.0
openshift_pkg_version=-3.11.0
openshift_image_tag=v3.11.0
openshift_service_catalog_image_version=v3.11.0
template_service_broker_image_version=v3.11
osm_use_cockpit=true

# put the router on dedicated infra1 node
openshift_master_cluster_method=native
openshift_master_default_subdomain=apps.openshift.com
openshift_public_hostname=master.openshift.com


:wq! (save and exit)
```
11) Use the below ansible playbook command to check the prerequisites to deply OpenShift Cluster on master Node:
******************************************************************************************************************
```
ansible-playbook -i ~/inventory.ini playbooks/prerequisites.yml
```
12) Once prerequisites completed without any error use the below ansible playbook to Deploy OpenShift Cluster on master Node:
**********************************************************************************************************************
```
ansible-playbook -i ~/inventory.ini playbooks/deploy_cluster.yml
```
13) Once the Installation is completed, change the following to use LDAP services, if not continue login with "oc login -u system:admin"  and create new users. 

Open /etc/origin/master/master-config.yaml file and do the following changes:
*********************************************************************************************************
```
 identityProviders:
  - name: ldap_authentication
    challenge: true
    login: true
    mappingMethod: claim
    provider:
      apiVersion: v1
      kind: LDAPPasswordIdentityProvider
      attributes:
        id:
        - sAMAccountName
        email:
        - mail
        preferredUsername:
        - sAMAccountName
      bindDN: ""
      bindPassword: ""
      baseDN: OU=enterprise,DC=fnc,DC=net,DC=local
      insecure: true
      url: "ldap://ldap.tx.fnc.fujitsu.com/OU=enterprise,DC=fnc,DC=net,DC=local?sAMAccountName" 


:wq! (save and exit)
```
14) To login as admin
```
oc login -u system:admin
```
15) Verifying Multiple etcd Hosts:
************************************
On a master host, verify the etcd cluster health, substituting for the FQDNs of your etcd hosts in
the following:
```
yum install etcd -y

etcdctl -C https://master.openshift.com:2379 --ca-file=/etc/origin/master/master.etcd-ca.crt --cert-file=/etc/origin/master/master.etcd-client.crt --key-file=/etc/origin/master/master.etcd-client.key cluster-health
```

16) Use below command to list the projects, pods, nodes, Replication Controllers, Services and Deployment Config.
*****************************************************************************************************************
```
oc whoami

oc get projects
NAME                    DISPLAY NAME   STATUS
default                                Active
kube-public                            Active
kube-system                            Active
logging                                Active
management-infra1                      Active
openshift                              Active
openshift-infra1                       Active
openshift-node                         Active
openshift-web-console                  Active

oc get nodes
NAME                   STATUS    ROLES     AGE       VERSION
infra.openshift.com    Ready     infra     7d        v1.11.0+d4cacc0
master.openshift.com   Ready     master    7d        v1.11.0+d4cacc0
node1.openshift.com    Ready     compute   7d        v1.11.0+d4cacc0
node2.openshift.com    Ready     compute   7d        v1.11.0+d4cacc0

oc get pod --all-namespaces
NAMESPACE               NAME                          READY     STATUS    RESTARTS   AGE
default                 docker-registry-1-krmpx       1/1       Running   0          15m
default                 registry-console-1-z9sbd      1/1       Running   0          14m
default                 router-1-q7g5v                1/1       Running   0          15m
openshift-web-console   webconsole-5f649b49b5-hr9dq   1/1       Running   0          14m

oc get rc
NAME                 DESIRED   CURRENT   READY     AGE
docker-registry-1    1         1         1         15m
registry-console-1   1         1         1         14m
router-1             1         1         1         16m

oc get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
docker-registry    ClusterIP   172.30.34.127    <none>        5000/TCP                  16m
kubernetes         ClusterIP   172.30.0.1       <none>        443/TCP,53/UDP,53/TCP     35m
registry-console   ClusterIP   172.30.62.197    <none>        9000/TCP                  15m
router             ClusterIP   172.30.141.143   <none>        80/TCP,443/TCP,1936/TCP   16m

oc get dc
NAME               REVISION   DESIRED   CURRENT   TRIGGERED BY
docker-registry    1          1         1         config
registry-console   1          1         1         config
router             1          1         1         config
```

# Kubernetes with kubespray at home

With kubespray you can deploy a high available Production Ready Kubernetes Cluster. 

Kubespray can be deployed on AWS, GCE, Azure, OpenStack, vSphere, Oracle Cloud Infrastructure (Experimental), or Baremetal.

With this guide you can setup a k8s cluster on your desktop using calico network plugin.

## Install vagrant with host-manager plugin

```console
$ wget -c https://releases.hashicorp.com/vagrant/2.0.3/vagrant_2.0.3_x86_64.deb
$ sudo dpkg -i vagrant_2.0.3_x86_64.deb
$ vagrant plugin install vagrant-hostmanager
```

## Create guest machines
```console
$ cd kubespray/
$ vagrant up

```
This command creates and configures guest machines according to the Vagrantfile. 
This Vagrantfile setups 4 vm. The first one with name *_control_* is used for the 
setup of k8s cluster with ansible. When ready control will have ansible 2.5 installed 
and kubestray repo downloaded at /vagrant/kubespray. The 3 other guest machines will 
be used as k8s nodes.

| node_name | address         | role             |
|-----------|-----------------|------------------|
| control   | 192.168.100.101 | runs ansible     |
| node1     | 192.168.100.102 | k8s master, node |
| node2     | 192.168.100.103 | k8s master, node |
| node3     | 192.168.100.104 | k8s node         |

```console
$ # check vm status
$ vagrant status
Current machine states:

control                   running (virtualbox)
node1                     running (virtualbox)
node2                     running (virtualbox)
node3                     running (virtualbox)
```

## Configure setup

Connect to *control* vm.
```console
$ vagrant ssh control
```
Install dependencies from _requirements.txt_ .
```console
vagrant@control:/vagrant$ cd /vagrant/kubespray/
vagrant@control:/vagrant/kubespray$ sudo pip install -r requirements.txt
```

Copy _inventory/sample_ as _inventory/mycluster_ .
```console
vagrant@control:/vagrant/kubespray$ cp -rfp inventory/sample inventory/mycluster
```

Update Ansible inventory file with inventory builder
```console
vagrant@control:/vagrant/kubespray$ declare -a IPS=(192.168.100.102 192.168.100.103 192.168.100.104)
```

Get extra-vars.json which ovewrites variables from _inventory/mycluster/group_vars/all/all.yml_ and 
_inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml_
```console
vagrant@control:/vagrant/kubespray$ cp ../extra-vars.json  .
```

Ordet the k8s cluster setup
```console
vagrant@control:/vagrant/kubespray$ ansible-playbook -i inventory/mycluster/hosts.ini --extra-vars="@extra-vars.json" --become --become-user=root cluster.yml
```

When finished ander artifacts/ you can find kubectl and admin.conf which is a # Make a copy of kubeconfig on the host that runs Ansible.
```console
vagrant@control:/vagrant$ sudo mv kubectl /usr/bin/
vagrant@control:/vagrant/kubespray$ export KUBECONFIG=/vagrant/kubespray/inventory/mycluster/artifacts/admin.conf
```

Order some kubectl commands
```console
vagrant@control:/vagrant$ kubectl get nodes
 NAME    STATUS   ROLES         AGE    VERSION
 node1   Ready    master,node   159m   v1.13.0
 node2   Ready    master,node   157m   v1.13.0
 node3   Ready    node          157m   v1.13.0

vagrant@control:/vagrant$ # check nginx ingress pods 
vagrant@control:/vagrant$ kubectl get pods -n ingress -o wide
  NAME                             READY   STATUS    RESTARTS   AGE   IP                NODE    NOMINATED NODE   READINESS GATES
  ingress-nginx-controller-672n2   1/1     Running   0          66m   192.168.100.103   node2   <none>           <none>
  ingress-nginx-controller-spgpl   1/1     Running   0          66m   192.168.100.102   node1   <none>           <none>
```

Expose nginx-ingress with `node port` at `port 80`.
```console
vagrant@control:/vagrant$ kubectl create -f nginx-ingress-service.yaml
vagrant@control:~$ kubectl get svc -n ingress
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
default-backend   ClusterIP   10.233.45.86   <none>        80/TCP      3h1m
ingress-nginx     NodePort    10.233.3.141   <none>        80:80/TCP   110m

```

Deploy an apache server at namespace `webservers`
```console
vagrant@control:/vagrant$ kubectl create namespace webservers
vagrant@control:/vagrant$ kubectl create -f apache-deployment.yml 
vagrant@control:/vagrant$ kubectl get pods -n webservers
NAME                                 READY   STATUS    RESTARTS   AGE
apache-deployment-6c8b6dc795-qn2rh   1/1     Running   0          1m
```
Expose deployment with a `ClusterIp` service
```console
vagrant@control:/vagrant$ kubectl create -f apache-deployment.yml
vagrant@control:/vagrant$ kubectl get svc -n webservers
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
apache-service   ClusterIP   10.233.37.239   <none>        80/TCP    1m
```
Create an ingress rule for apache
```console
vagrant@control:/vagrant$ kubectl create -f apache-ingress.yaml
vagrant@control:/vagrant$ kubectl describe ingress apache -n webservers
Name:             apache
Namespace:        webservers
Address:          
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host                  Path  Backends
  ----                  ----  --------
  apache.mycluster.k8s  
                        /*   apache-service:80 (<none>)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  2m8s  nginx-ingress-controller  Ingress webservers/apache
  Normal  CREATE  2m8s  nginx-ingress-controller  Ingress webservers/apache
  Normal  UPDATE  83s   nginx-ingress-controller  Ingress webservers/apache
  Normal  UPDATE  83s   nginx-ingress-controller  Ingress webservers/apache
```

Check that ingress rule works:
```console
vagrant@control:/vagrant$ curl -sS 192.168.100.103 --header "Host: apache.mycluster.k8s"
<html><body><h1>It works!</h1></body></html>
```

## Setup dnsmasq 
Setup dnmasq to resolve requests to domain mycluster.k8s to k8s cluster.
```console
vagrant@control:/vagrant$ sudo apt-get install dnsmasq
vagrant@control:/vagrant$ sudo vi /etc/dnsmasq.d/k8s.conf
```
and add the content
```console
bind-interfaces
listen-address=127.0.0.1
address=/mycluster.k8s/192.168.100.102
```
restart dnmasq and verify that all works
```console
vagrant@control:/vagrant$ sudo systemctl restart dnsmasq
vagrant@control:/vagrant$  dig +noall +answer apache.mycluster.k8s  @127.0.0.1
apache.mycluster.k8s.	0	IN	A	192.168.100.102
```
now make the `GET` request without the Host parameter in Header 
```console
vagrant@control:/vagrant$ curl apache.mycluster.k8s
<html><body><h1>It works!</h1></body></html>

```

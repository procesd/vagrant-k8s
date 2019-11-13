# Kubernetes cluster in vagrant
## IMPORTANT
This is a learning exercise, you will get a k8s cluster but the idea is to check the details and follow the steps. If you just want to get a running cluster I suggest you to use `kubespray` (https://github.com/kubernetes-sigs/kubespray), it will be way better maintained than this project.

## Description
Create a kubernetes cluster in virtual box
We use packer to create the base box (see https://github.com/procesd/packer-vagrant-k8s-base )
Then we will use vagrant to create the VMs , the next step will be Ansible to provision them.

## Design decisions / assumptions
### Single plane control
(fancy way of saying no redundancy on master?) following https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

Having HA on masters for a dev/learning environment would require (at least) two more masters and ideally another 2 servers for a HA load balancer using HAproxy. My laptop would probably melt.

For production obviously you need HA, see https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/  for the setup

### Pod network
Flannel, should we go with Weave Net? it seems more production ready as it has `Network Policies`, `Ingress/Egress Policies`, encryption and commercial support (Live hack: Always make sure your outage is also the problem of a subject matter expert!). Also mesh(???)

## vagrant-Hostsmanager
see https://github.com/devopsgroup-io/vagrant-hostmanager

Simplify your ops adding the nodes to your host file so you don't have to remember IPs
We need to add the plugin `vagrant-hostsmanager` running the following command:
`vagrant plugin install vagrant-hostsmanager`

In order for the hosts to be added to /etc/hosts we need to define `master.vm.network` and `master.vm.hostname` in the Vagrantfile

IE: We generate several master (default 1) with the following code and assign this value
```
(1..MASTERS).each do |i|
  config.vm.define "master-#{i}" do |master|
    master.vm.network "private_network", ip: "192.168.5.10#{i}"
    master.vm.hostname = "master-#{i}"
  end
end
```

## Ansible
### Ansible as vagrant provisioner
Usually vagrant runs the ansible scripts node by node with an implicit ``--limit <node>`` but that won't work for us as we need to interact with master(s) when configuring the worker nodes.

In order to execute it:
* only once
* running in all nodes in parallel
* sharing info from different hosts
we use this pattern from https://www.vagrantup.com/docs/provisioning/ansible.html#tips-and-tricks

```
....
(1..NODES).each do |i|
  config.vm.define "node-#{i}" do |node|
    ....
    if i == NODES
       config.vm.provision "ansible" do |ansible|
         ansible.limit = "all"
         ansible.playbook = "ansible/vagrant.yml"
         ansible.groups = ansible_groups
       end #config
    end

```

Vagrant will create the ansible inventory file in `.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory`. In our case, we need to define the groups [master] and [node] setting
the variable `ansible_groups`
```
...
MASTERS=1
NODES=3
ansible_groups = {
  "master" => [
    "master-[1:#{MASTERS}]"
  ],
  "node" => [
    "node-[1:#{NODES}]"
  ]
}
...
```

If you have to make changes to the playbook you should be able to use
`ansible-playbook --user vagrant -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory ansible/vagrant.yml`

### Provisioning Kubernetes

#### Step 1 Creating the control plane  with `kubeadm init <options>`
As we have 2 network interfaces (enp0s3 and enp0s8) we want to make sure that kubeadm uses the proper one for the cluster.
enp0s3 is for a NAT interface , giving us access to internet but not accessible from hosts, we don't know the IP before starting up the server as it's given by DHCP
enp0s8 is a Host-only network card so we can access the apps listening in this interface. We have fixed the IP in the Vagrantfile (master will use 192.168.5.101) and we want to use this for the cluster. As per default Kubeadm uses the interface in the same network as the gateway (enp0s3) so we want to make explicit that it should use enp0s8 with the option `--apiserver-advertise-address`

We will also use Flannel as a network pod (details below) which require to pass the cidr network explicitly to  

 so the final command will be:
`sudo kubeadm init --apiserver-advertise-address=192.168.5.101 --pod-network-cidr=10.244.0.0/16`

To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


It will give us the command used to join the worker nodes
`kubeadm join 192.168.5.101:6443 --token <TOKEN>  --discovery-token-ca-cert-hash sha256:<HASH>`

kubeadm join 192.168.5.101:6443 --token koc4fq.29wo7ie3f4vokdjk \
    --discovery-token-ca-cert-hash sha256:bfbe516fc52b514034578bc2493c72f0b31a2ab9342f97acf340fc1ee6bafd17


#### Step 2: Adding the worker nodes
Just ssh into each node and run the line that kubeadm gave us
`sudo kubeadm join 192.168.5.101:6443 --token 467f27.usqpruw3rx356bzz \
    --discovery-token-ca-cert-hash sha256:52e5ab52a984534840d3de959c66a15107c910b2bf627cee845dc803dd84f6d5`


#### Step 3 Installing a pod network add-on
Even if the hosts can ping each other the pods will be using a kubernetes "internal" network/s, this will be created by a network plugin.

Check that coreDNS (why others like etcd-master-1 are running, what's the difference between ?) willnot run without a pod network installed
kube-apiserver-master-1
```
vagrant@master-1:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-5644d7b6d9-z9kvn           0/1     Pending   0          10m
kube-system   coredns-5644d7b6d9-zj824           0/1     Pending   0          10m
kube-system   etcd-master-1                      1/1     Running   0          9m16s
...
```

We passed the CIDR for Flannel ( does it need to be exactly that network?) so in order to add this pod network we will install the Flannel deployments

NOTE: We need to pass bridged IPv4 traffic to iptables’ chains.This is allowed or denied in the kernel. Check that the kernel option `net.bridge.bridge-nf-call-iptables` is set to 1, this is mapped to the proc filesystem in `/proc/sys/net/bridge/bridge-nf-call-iptables` you can cat the file toi see the value, that should be 1. In case it's not you must run `sysctl net.bridge.bridge-nf-call-iptables=1`.
In order to make this change permanent you can edit /etc/sysctl.conf and set the option there.

`wget https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml`

### Edit the default interface for the kube-flannel containers
( with help from https://stackoverflow.com/questions/47845739/configuring-flannel-to-use-a-non-default-interface-in-kubernetes )

Flannel choose the first iface, which is not the one used by k8s so there is no connectivity
we have to add the option `--iface=enp0s8` to the container definition under the Daemonset for our architecture (amd64 in vagrant but can be different,IE in raspberry pi clusters should be arm64)
```
containers:
- name: kube-flannel
  image: quay.io/coreos/flannel:v0.11.0-amd64
  command:
  - /opt/bin/flanneld
  args:
  - --ip-masq
  - --kube-subnet-mgr
  resources:
  ...
  ```

and add `--iface=enp0s8` leaving it as
```
containers:
- name: kube-flannel
  image: quay.io/coreos/flannel:v0.11.0-amd64
  command:
  - /opt/bin/flanneld
  args:
  - --ip-masq
  - --kube-subnet-mgr
  - --iface=enp0s8
  resources:
  ...
```

Then apply the config
  `kubectl apply -f kube-flannel.yml`

After that CoreDNS should be running in the node-1
From master-1 (control-plane)
```
vagrant@master-1:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-5644d7b6d9-z9kvn           1/1     Running   0          10m
kube-system   coredns-5644d7b6d9-zj824           1/1     Running   0          10m
kube-system   etcd-master-1                      1/1     Running   0          9m16s
...
```

Step 3 Adding the dashboard
https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/


`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml`

Check the pods have started
```
vagrant@master-1:~$ kubectl get pods -o wide -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE    IP           NODE     NOMINATED NODE   READINESS GATES
dashboard-metrics-scraper-566cddb686-95h6h   1/1     Running   0          105s   10.244.2.2   node-2   <none>           <none>
kubernetes-dashboard-7b5bf5d559-6fbk8        1/1     Running   0          105s   10.244.3.2   node-3   <none>           <none>
```
#### To access it:
in master-1 `kubectl proxy`
Pretty funky tunnel, ignoring vagrant ssh and using host-only interface
`ssh vagrant@master-1 -i <VAGRANT_DIR>/.vagrant/machines/master-1/virtualbox/private_key -L8001:localhost:8001`

Access it from your laptop
go to your web browser
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/


#### Authentication
This is for a dev cluster,  as this commands are giving cluster admin permissions to the dashboard user

1 Create a  new service account
`kubectl create serviceaccount cluster-admin-dashboard-sa`

2 Bind ClusterAdmin role to the service account
kubectl create clusterrolebinding cluster-admin-dashboard-sa \
  --clusterrole=cluster-admin \
  --serviceaccount=default:cluster-admin-dashboard-sa

3 Get  the token
First we get the name of the service (with the extra random chars at the end)
```
vagrant@master-1:~$ kubectl get secret
NAME                                     TYPE                                  DATA   AGE
cluster-admin-dashboard-sa-token-zsh4l   kubernetes.io/service-account-token   3      4m39s
default-token-jswvg                      kubernetes.io/service-account-token   3      52m
```

Then we can get the token
```
vagrant@master-1:~$ kubectl describe secret cluster-admin-dashboard-sa-token-zsh4l
Name:         cluster-admin-dashboard-sa-token-zsh4l
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: cluster-admin-dashboard-sa
              kubernetes.io/service-account.uid: 5e5fff0a-c6e1-4582-8fb7-a7550b5155c0

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InR6Y0dyMGRHMnNNUWxMeTNOdnM4akRtbnROUXJkWEJPYzJ1WGNQMy1PNFUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImNsdXN0ZXItYWRtaW4tZGFzaGJvYXJkLXNhLXRva2VuLXpzaDRsIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImNsdXN0ZXItYWRtaW4tZGFzaGJvYXJkLXNhIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNWU1ZmZmMGEtYzZlMS00NTgyLThmYjctYTc1NTBiNTE1NWMwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6Y2x1c3Rlci1hZG1pbi1kYXNoYm9hcmQtc2EifQ.Fr5S2Ni46XBd0YVr2wg756_jDMRuFXOuaA0IAEouztjkuZOEISJC1AdM200GVAlwdSuZN3MsqcY39Pa2miYgeoh1mZvHe7RtuYeSwFIS0xwCpeBLzJVGKWw52NSY5rUosURdUR_1Yn52gXWUJTDmaWuKHL1BpUFpMKWrcGNftrcJHp8bZodhS-5zcCTi4gGUS8YMUxSYGjg1GBvKZlPcWSvbU2Q2vCjORQIithV5sAnXRdpQOzh7Qy7UITTDAEqB-PZFYC5ly-TZI3H_vc2xnzNTf_GHFNEV7VU20s5O3MD0q1hwPFOVj9liOP4rrAvtwTeMkbzC317OBN46CsAfMg
```
4 Finally we copy paste the token in the web browser



Operations
Stop an existing cluster
Stop worker nodes
stop master nodes
Stop NFS or other filesystem/DB dependencies of the cluster (etcd if external?)


Start an existing clusters
Start master dependencies (NFS, external etcd, ??), start masters, start pods dependencies (dbs?) start nodes

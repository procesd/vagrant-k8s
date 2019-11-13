TODO:
reuse vagrant pubkey to be able to ssh directly or run ansible outside vagrant
config.ssh.private_key_path = "custom_key_file"
config.ssh.forward_agent = true

DONE: change to Hostsmanager, see https://github.com/devopsgroup-io/vagrant-hostmanager and update TODO

TODO: Manual setup to ansible playbook


TODO: add more masters? run something only on master-1, then add the others
to run the cluster initialization on only one master use `hosts: group_name[0]`
to run something on the other masters:  `hosts: group_name[1:]`
for the master to join existing cluster see https://blog.scottlowe.org/2019/08/15/reconstructing-the-join-command-for-kubeadm/ for


TODO: Centralized logging: https://medium.com/@maanadev/centralized-logging-in-kubernetes-d5a21ae10c6e

TODO: create manifests!
make a silly webapp
follow https://prefetch.net/blog/2019/10/16/the-beginners-guide-to-creating-kubernetes-manifests/

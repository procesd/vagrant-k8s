# -*- mode: ruby -*-
# vi: set ft=ruby :

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
Vagrant.require_version ">= 2.2.5"


Vagrant.configure("2") do |config|
  config.vm.box = "procesd/ubuntu18.04-k8s"
  config.vm.box_version = "0.0.5"
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  config.vm.provider "virtualbox" do |vb|
    vb.linked_clone = true
  end

  (1..MASTERS).each do |i|
    config.vm.define "master-#{i}" do |master|
      master.vm.network "private_network", ip: "192.168.5.10#{i}"
      master.vm.hostname = "master-#{i}"
      if Vagrant.has_plugin?("vagrant-vbguest")
        master.vbguest.auto_update = false
        master.vm.synced_folder '.', '/vagrant', disabled: true
      end #disable vbguest additions
    end # loop create masters
  end #vagrant configure

  (1..NODES).each do |j|
    config.vm.define "node-#{j}" do |node|
      node.vm.network "private_network", ip: "192.168.5.11#{j}"
      node.vm.hostname = "node-#{j}"

      if Vagrant.has_plugin?("vagrant-vbguest")
        node.vbguest.auto_update = false
        node.vm.synced_folder '.', '/vagrant', disabled: true
      end #disable vbguest additions

      if j == NODES
         node.vm.provision "ansible" do |ansible|
           ansible.limit = "all"
           ansible.playbook = "ansible/vagrant.yml"
           #ansible.inventory = ["__GENERATED__", "ansible/inventory_groups"]
           ansible.groups = ansible_groups
         end #config
      end

    end #config.vm.define
  end #each

end # Vagrant.configure

# -*- mode: ruby -*-
# vi: set ft=ruby :

# README
#
# Getting Started:
# 1. vagrant plugin install vagrant-hostmanager
# 2. vagrant up
# 3. vagrant ssh
#
# This should put you at the control host
#  with access, by name, to other vms
Vagrant.configure(2) do |config|
    config.hostmanager.enabled = true
    config.vm.synced_folder "./", "/vagrant_data"

    config.vm.define "control", primary: true do |control|
        control.vm.box = "ubuntu/xenial64"
        control.vm.hostname = 'control'

	control.vm.network "private_network", ip: "192.168.100.101"
    	control.vm.provision "shell",
          inline: '
if [ ! -f "/home/vagrant/.ssh/id_rsa" ]; then
    ssh-keygen -t rsa -N "" -f /home/vagrant/.ssh/id_rsa
fi
cp /home/vagrant/.ssh/id_rsa.pub /vagrant/control.pub

cat <<-SSHEOF > /home/vagrant/.ssh/config
Host *
   StrictHostKeyChecking no
   UserKnownHostsFile=/dev/null
SSHEOF

chown -R vagrant:vagrant /home/vagrant/.ssh/

apt-get update --yes
apt-get install --yes software-properties-common
apt-add-repository --yes --update ppa:ansible/ansible-2.5
apt-get update --yes
apt-get install --yes ansible=2.5.13-1ppa~xenial
locale-gen en_US en_US.UTF-8
echo "LC_ALL=en_US.UTF-8" >> /etc/environment 
echo "LANG=en_US.UTF-8" >> /etc/environment 
apt-get install --yes python-pip
apt-get install --yes python-netaddr
git clone https://github.com/kubernetes-sigs/kubespray.git /vagrant/kubespray
git --git-dir /vagrant/kubespray/.git fetch origin 0d1be39a97d85436edf2ed0f1db250846c0ba504
git --git-dir /vagrant/kubespray/.git reset --hard FETCH_HEAD
'

        config.vm.provider "virtualbox" do |vb|
        #   Display the VirtualBox GUI when booting the machine
        #   vb.gui = true
        #   Customize the amount of memory on the VM:
            vb.customize ["modifyvm", :id, "--memory", 1024]
        end
    end

    config.vm.define "node1" do |node1|
        node1.vm.box = "ubuntu/xenial64"
        node1.vm.hostname = "node1"
        node1.vm.network "private_network", ip: "192.168.100.102"
        node1.vm.provision :shell, inline: 'cat /vagrant/control.pub >> /home/vagrant/.ssh/authorized_keys'
        node1.vm.provider "virtualbox" do |vb|
        #   Display the VirtualBox GUI when booting the machine
        #   vb.gui = true
        #   Customize the amount of memory on the VM:
            vb.customize ["modifyvm", :id, "--memory", 3072]
        end
    end

    config.vm.define "node2" do |node2|
        node2.vm.box = "ubuntu/xenial64"
        node2.vm.hostname = "node2"
        node2.vm.network "private_network", ip: "192.168.100.103"
        node2.vm.provision :shell, inline: 'cat /vagrant/control.pub >> /home/vagrant/.ssh/authorized_keys'
        node2.vm.provider "virtualbox" do |vb|
        #   Display the VirtualBox GUI when booting the machine
        #   vb.gui = true
        #   Customize the amount of memory on the VM:
            vb.customize ["modifyvm", :id, "--memory", 3072]
        end
    end

    config.vm.define "node3" do |node3|
        node3.vm.box = "ubuntu/xenial64"
        node3.vm.hostname = "node3"
        node3.vm.network "private_network", ip: "192.168.100.104"
        node3.vm.provision :shell, inline: 'cat /vagrant/control.pub >> /home/vagrant/.ssh/authorized_keys'
        node3.vm.provider "virtualbox" do |vb|
        #   Display the VirtualBox GUI when booting the machine
        #   vb.gui = true
        #   Customize the amount of memory on the VM:
            vb.customize ["modifyvm", :id, "--memory", 3072]
        end
    end
end


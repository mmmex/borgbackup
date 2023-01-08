# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "backup_server" do |srv|
    srv.vm.box = "centos/7"
    srv.vm.box_check_update = false
    srv.vm.hostname = "server"
    srv.vm.network "private_network", ip: "192.168.56.2"
    srv.vm.provider "virtualbox" do |vb|
      second_disk = "disk.vdi"
      unless File.exist?("disk.vdi")
        vb.customize ['createhd', '--filename', second_disk, '--size', 3 * 1024]
      end
      vb.customize ['storageattach', :id, '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', second_disk]
    end
  end

  config.vm.define "client" do |cli|
    cli.vm.box = "centos/7"
    cli.vm.box_check_update = false
    cli.vm.hostname = "client"
    cli.vm.network "private_network", ip: "192.168.56.3"
  end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "provision.yml"
  end
end

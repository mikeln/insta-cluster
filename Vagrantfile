# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

VAGRANTFILE_API_VERSION = "2"
Vagrant.require_version ">= 1.7.2"

COREOS_CHANNEL = "alpha"
COREOS_RELEASE = "current"
IMAGE_PATH = "data/tftpboot"
DATA_PATH = "data/"
MASTER_USER_DATA = File.expand_path(File.join(File.expand_path(File.dirname(__FILE__)), 'master.yaml'))
NODE_USER_DATA = File.expand_path(File.join(File.expand_path(File.dirname(__FILE__)), 'node.yaml'))

required_plugins = %w(vagrant-triggers vagrant-aws)

required_plugins.each do |plugin|
  need_restart = false
  unless Vagrant.has_plugin? plugin
    system "vagrant plugin install #{plugin}"
    need_restart = true
  end
  exec "vagrant #{ARGV.join(' ')}" if need_restart
end

Vagrant.configure(2) do |config|

  config.vm.define "master" do |master|

    master.vm.box = "http://#{COREOS_CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_RELEASE}/coreos_production_vagrant.box"
    master.vm.box_check_update = true

    master.ssh.insert_key = false

    master.vm.network "public_network", adapter: 2,  bridge: '', ip: "172.16.16.15"
    master.vm.network "private_network", adapter: 3, ip: "192.168.100.10" 

    master.vm.network "forwarded_port", guest: 2375, host: 2375

    master.vm.hostname = "master"
    
    master.vm.synced_folder ".", "/vagrant", disabled: false, type: "nfs", nfs_udp: true, mount_options: ['rsize=32768', 'wsize=32768', 'nolock']
    
    master.vm.provider :virtualbox do |mvb, override|
      mvb.memory = 4048
      mvb.cpus = 2
      mvb.customize ["controlvm", :id, "nicpromisc2", "allow-all"]
    end

    if File.exists?(MASTER_USER_DATA)
      master.vm.provision :file, :source => MASTER_USER_DATA, :destination => "/tmp/vagrantfile-user-data"
      master.vm.provision :shell, :privileged => true,
      inline: <<-EOF
        mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/
      EOF
    end

    # Download and install dependencies
    config.trigger.after [:up] do
      # Download the corresponding CoreOS image files for the the TFTP boot server
      # Will not download images if they are not newer then the existing files
      system "wget -N -P #{IMAGE_PATH} http://#{COREOS_CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_RELEASE}/coreos_production_pxe.vmlinuz"
      system "wget -N -P #{IMAGE_PATH} http://#{COREOS_CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_RELEASE}/coreos_production_pxe_image.cpio.gz"

      # Download and extract the docker registry and the local registry docker images
      # Will not download files if they are not newer then the existing files
      system "wget -N -P #{DATA_PATH} https://s3-us-west-2.amazonaws.com/insta-cluster/docker-registry.tar"
      system "wget -N -P #{DATA_PATH} https://s3-us-west-2.amazonaws.com/insta-cluster/private.tar.gz"
      system "tar -zxf #{DATA_PATH}private.tar.gz -C #{DATA_PATH}"

      # Download the required version for kubectl and docker compose for this cluster service deployemnt
      system "wget -N -P bin https://s3-us-west-2.amazonaws.com/insta-cluster/kubectl"
      system "wget -N -P bin https://s3-us-west-2.amazonaws.com/insta-cluster/docker-compose"

      # Make all files executable
      system "chmod -R +x bin"
    end
  end

  config.vm.define "node-01" do |node|
    node.vm.box = "http://#{COREOS_CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_RELEASE}/coreos_production_vagrant.box"
    node.vm.box_check_update = true

    node.ssh.insert_key = false

    node.vm.network "public_network", adapter: 2, bridge: '', ip: "172.16.16.16"
    node.vm.network "private_network", adapter: 3, ip: "192.168.100.11" 

    node.vm.hostname = "node-01"

    node.vm.synced_folder ".", "/vagrant", disabled: false, type: "nfs", nfs_udp: true, mount_options: ['rsize=32768', 'wsize=32768', 'nolock']
    
    node.vm.provider :virtualbox do |nvb, override|
      nvb.memory = 4048
      nvb.cpus = 2
      nvb.customize ["controlvm", :id, "nicpromisc2", "allow-all"]
    end

    if File.exists?(NODE_USER_DATA)
      node.vm.provision :file, :source => NODE_USER_DATA, :destination => "/tmp/vagrantfile-user-data"
      node.vm.provision :shell, :privileged => true,
      inline: <<-EOF
        mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/
      EOF
    end
  end
end

# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
  :lvm => {
        :box_name => "centos/8",
        :box_version => "0",
        :ip_addr => '192.168.56.10',
    :disks => {
        :sata1 => {
            :dfile => home + '/VirtualBox VMs/sata1.vdi',
            :size => 10240,
            :port => 1
        }
    }
  },
}

Vagrant.configure("2") do |config|
    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
            box.vm.provider :virtualbox do |vb|
                vb.gui = true
		        vb.customize ["modifyvm", :id, "--memory", "512"]
                needsController = false
                boxconfig[:disks].each do |dname, dconf|
                    unless File.exist?(dconf[:dfile])
                        vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                        needsController =  true
                    end
  
                end
                if needsController == true
                    vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                    boxconfig[:disks].each do |dname, dconf|
                        vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                    end
                end
            end
  
        box.vm.provision "shell", inline: <<-SHELL            
            sudo -i
            mkdir -p ~root/.ssh
            cp ~vagrant/.ssh/auth* ~root/.ssh
            sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
            sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
            yum install -y mdadm smartmontools hdparm gdisk lvm2 nano xfsdump wget
          SHELL
        end
    end
  end

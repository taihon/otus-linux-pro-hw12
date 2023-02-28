# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
  :selinux => {
        :box_name => "centos/7",
        :box_version => "2004.01",
        :ip_addr => '192.168.11.102',
    :disks => {
    }
  },
}

Vagrant.configure("2") do |config|

    config.vm.box_version = "2004.01"
    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
  
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
  
            box.vm.network "forwarded_port", guest: 4881, host: 4881
  
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
  
            box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "256"]
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
        box.vm.provision "shell", inline: <<-'SHELL'
          mkdir -p ~root/.ssh
          cp ~vagrant/.ssh/auth* ~root/.ssh
          yum install -y epel-release
          yum install -y nginx selinux-policy-devel
          sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
          sed -ir 's/listen\s\{1,\}80;/listen\t4881;/' /etc/nginx/nginx.conf
          systemctl start nginx
          systemctl status nginx
          ss -tlpn|grep 4881
          SHELL
    	end
  	end
end
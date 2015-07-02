# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.

$script = <<SCRIPT
if [ "$(( $(cat /proc/swaps|wc -l) - 1 ))" -eq 1 ]; then
sudo mkdir -p /mnt/swap
cd /mnt/swap/
sudo dd if=/dev/zero of=swapfile bs=1M count=1536 
sudo mkswap swapfile 
sudo swapon swapfile
fi
SCRIPT

Vagrant.configure(2) do |config|

  config.vm.box = "centos-6.6-virtualbox"
  config.vm.box_url = "http://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_centos-6.6_chef-provisionerless.box" 
  
  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024"]
  end
   
  # Application server.
  config.vm.define "wcs-server" do |app|
    app.vm.hostname = "wcs.192.168.60.5.xip.io"
    app.vm.network :private_network, ip: "192.168.60.5"
    app.vm.provision "shell", inline: $script
    app.vm.network "forwarded_port", guest: 8080, host: 8080, auto_correct: true
    app.vm.network "forwarded_port", guest: 1521, host: 1521, auto_correct: true      
    app.vm.network "forwarded_port", guest: 9090, host: 9090, auto_correct: true
    app.vm.provision "ansible" do |ansible|
      ansible.playbook = "wcs.yml"
      ansible.inventory_path = "hosts"
      ansible.verbose = "extra"
      ansible.limit = 'all'      
    end
  end  
  
end

# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.



Vagrant.configure(2) do |config|

  config.vm.box = "wcs-centos-6.6-virtualbox"
  #config.vm.box_url = "http://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_centos-6.6_chef-provisionerless.box" 
  
  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024"]
  end
   
  # Application server.
  config.vm.define "wcs-server" do |app|
    app.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
  end    
  
end

# Part 1: Creating Web Center Sites Environments
For this, the first tutorial in our Continuous Delivery series, we’re starting right at the beginning with a look at creating environments to support Web Center Sites development.

We’ll look at creating environments that are as alike as we can make them, up through the stack - from development through test up to production environments. We’ll also look at automating this process because you know…

AUTOMATE ALL THE THINGS.

In more detail we’ll look first at creating a Web Center Sites development environment. To do that we’ll need to automate the installation and configuration of Web Center Sites. In order to install WCS we also need to install and configure some supporting software. 

This includes:
- An operating system (Centos 6.6)
- An application server (Tomcat 7.0.62)
- A database (Oracle 11g XE)

To make all of that repeatable, we’ll be encapsulating all the logic by using a Configuration Management tool called Ansible.

Finally we’ll use a tool called packer.io to create machine images that make it quick and easy for our developers to get a development environment up and running. Packer gives us two key benefits. The first is that this process of producing a development environment is repeatable and the second is that the process of producing machine images can be extended to a range of output formats, more of that later.

Before we get going it’s worth talking a little about why we would want to do any of these things. I mean this setup takes a little bit of work so let’s start with a business case of sorts.

- **Reducing the capacity for surprise**  
	Let me start with a question: How many times as a developer have you deployed code into a test environment only for it to fail because the test environment is configured differently in some way? - perhaps a different database, or a different version of the application server? perhaps even a different patch version of Web Center Sites. Even worse, how many time has that happened in production?  
	We can mitigate the risk of these sorts of problems by trying to make our development and test environments as ‘Production Like’ as possible, this means the same OS, the same versions of supporting applications configured in the same way. We should also be able to deploy our code in the same kind of way as we do in production so we know better, how that process is going to behave.

- **Avoiding drift**
	Managing environments is hard, managing them manually is really hard. Martin Fowler talks about what he calls SnowFlake servers, these are environments that have been hand crafted and require tweaking and updating manually. The problem here is when configuration needs to change, whether this is in production or development, firstly making those changes manually has an associated risk that somebody gets it wrong and secondly the more servers you have to update, the greater the risk that you forget one or that the change isn’t applied in exactly the same way.  Once this happens the configuration of your environments starts to drift. I see this problem particularly and often between environment tiers like Test and Production. 

There are a number of good ways to solve the problems described above, we’ll focus on two of them - describing the configuration of our environments using recipes and using those recipes to create golden masters.

> A good way of ensuring you are avoiding snowflakes is to use PhoenixServers. Using version-controlled recipes to define server configurations is an important part of Continuous Delivery.
> Martin Fowler (http://martinfowler.com/bliki/SnowflakeServer.html)

## Configuration Management Tools
There are a number of well used Configuration Management tools available now - Puppet and Chef get a lot of mindshare but for this tutorial I wanted to look at another tool called Ansible - http://www.ansible.com. The thing I like best about Ansible is its love of the shell. As much as I like writing Ruby, if there’s something I want automate in an environment, I’ll typically turn to bash first. Ansible feels to me like an extension of shell scripting and because of that the process of building Ansible recipes (or playbooks in fact) feels like it involves less cognitive friction. YMMV however and the basic premise of what follows is as easily implemented using Puppet or Chef. 


## Getting Started
Before we build our development environment we’ll need to firstly install some software on our local machine and secondly download some software from Oracle - this is because of Oracle’s license restrictions.

On your local machine:
- Install VirtualBox - [https://www.virtualbox.org/wiki/Downloads][1]
- Install Vagrant - [http://www.vagrantup.com/downloads][2]
- Install Packer - [https://www.packer.io/intro/getting-started/setup.html][3]

Disclosure - I installed vagrant using the provided installer for Mac OS X and I installed Packer using HomeBrew.

Download Oracle Software
- Web Center Sites - [http://www.oracle.com/technetwork/middleware/webcenter/sites/downloads/index.html][4]
- Oracle 11g XE - [http://www.oracle.com/technetwork/database/database-technologies/express-edition/downloads/index.html][5] (here you want the x64 installer for Linux)
- Oracle JDBC Driver - [http://www.oracle.com/technetwork/apps-tech/jdbc-112010-090769.html][6]

Next, somewhere on your local filesystem clone our starter repo from github - `git clone https://github.com/Manifesto-Digital/wcs-env-tutorial` and then move into that directory.

It’s worth noting that this repository has a number of branches included that will help you follow along with the tutorial - you don’t have to use these branches but hopefully they give you something to compare to if you’re following along yourself.

### Introduction to Vagrant
Inside our starter project are a number of folders and files but let’s start with looking at vagrant first. If you want you can change the branch of your local git repo to this starting point, or you can start by just copying the vagrant file below

	git checkout 1-intro-to-vagrant

The following code block shows what your Vagrantfile should look like at this point.

	# -*- mode: ruby -*-
	# vi: set ft=ruby :
	
	# All Vagrant configuration is done below. The "2" in Vagrant.configure
	# configures the configuration version (we support older styles for
	# backwards compatibility). Please don't change it unless you know what
	# you're doing.
	
	Vagrant.configure(2) do |config|
	
	  config.vm.box = "opscode_centos-6.6"
	  config.vm.box_url = "http://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_centos-6.6_chef-provisionerless.box" 
	
	  config.vm.provider :virtualbox do |vb|
	vb.customize ["modifyvm", :id, "--memory", "1024"]
	  end
	
	  # Application server.
	  config.vm.define "wcs-server" do |app|
	    app.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
	  end    
	end

Most of this is fairly vanilla but it’s worth pointing out a couple of things. 

The first is that for now we’re using Vagrant Box provided by someone else. Once we get to looking at Packer we’ll replace it but for now we’re using a Centos 6.6 box from the Bento project. Bento [https://github.com/chef/bento][7] is a project from the test engineers at Chef to provide Vagrant test environments and therefore provides a whole range of good starter Vagrant boxes.

The second is that we’re changing the RAM our VM will startup with in preparation for the software that we’ll look to add later.

And thirdly we’re setting up some networking for our box - we’re  forwarding port 8080 on our local machine to port 80 on the Vagrant box.

Let’s start by bring our vagrant box up - from your command line issue the following command

	vagrant up

If this is the first time you’ve issued the command you should see Vagrant start by downloading the referenced opscode box. Once that’s complete you should be able to connect to the box by issuing the following command 

	vagrant ssh

This will take you into a shell inside your vagrant box as the vagrant user. This user has sudo access so if you want to perform a task as the root user, you can.

Let’s start by doing something temporary to test our networking setup. From within the vagrant box issue the following command

	sudo yum -y install httpd && sudo service httpd start

This should install and start an instance of the Apache Web Server which you can validate by opening a web browser of navigating to http://localhost:8080 where you should see the Apache test page. 

Finally lets clean up by destroying our vagrant box

	vagrant destroy

### Introduction to Ansible
So now that we have the basis of our Development environment (an OS basically) we need to start installing software in our Vagrant box. To do that we’re using Ansible to write some playbooks. These playbooks include the instructions for installing our software in a kind of recipe format. Again you can either checkout the branch that maps to where we are in the process or you can follow along manually.

	git checkout 2-intro-to-ansible

The first thing you’ll notice is that we’ve made some changes to our Vagrantfile which now looks like this:

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
 

There’s quite a bit of change here so let’s take it one bit at a time. 

#### Temporary Horrible swap Hack

To start with we have a a little bit of inline provisioning that looks like this. I won’t spend too much time talking about it as we’re going to remove it later, however it’s here to help us get Oracle XE installed as the DB installer gets a little upset if we don’t have enough swap space in our VM. Don’t worry though we’ll fix things up properly when get to using packer.

	$script = <<SCRIPT
	if [ "$(( $(cat /proc/swaps|wc -l) - 1 ))" -eq 1 ]; then
	sudo mkdir -p /mnt/swap
	cd /mnt/swap/
	sudo dd if=/dev/zero of=swapfile bs=1M count=1536 
	sudo mkswap swapfile 
	sudo swapon swapfile
	fi
	SCRIPT

Next we’ve added some extra network configuration to make things a bit easier for our developers. 

Firstly we’ve added some more forwarded ports. These are for making access to the DB a bit easier (1521 and 8080 is now used for the database admin webapp) and we’ve added another new mapping for our application server tomcat (9090).

Secondly we’ve changed the network configuration so that we’re using a local private network and we’ve added a hostname that maps a fqdn to our local ip address. To do this we’re using a service called [http://xip.io]() that provides DNS mapping to any ip address as long as you use their format.

Finally we’ve added in a provisioning block that kicks of our ansible playbooks. The provisioning looks like this:

	  app.vm.provision "ansible" do |ansible|
	      ansible.playbook = "wcs.yml"
	      ansible.inventory_path = "hosts"
	      ansible.verbose = "extra"
	      ansible.limit = 'all'      
	  end

Simply it tells vagrant where to find some important ansible files like the main playbook and our inventory.

Our main ansible playbook is called wcs.yml and sits in the main directory. Lets have a quick look at it:

	---
	# file: wcs.yml
	- hosts: webservers
	  sudo: true
	  roles:
	    - common
	    - {role: oracle, tags: ["oracle"]}
	    - {role: tomcat, tags: ["tomcat"]}
	    - {role: web-center-sites, tags: ["wcs"]}      

In essence we’re saying we want to run the roles oracle, tomcat and web-center-sites in that order.


### Installing Oracle XE
To start with we’re going to install Oracle XE. Before we do so we need to place the oracle rpm archive we downloaded right at the beginning and place it within the roles/oracle/files directory.

The filesystem layout for our ansible role should like this:

	- oracle
	   - files
	       + oracle-xe-11.2.0-1.0.x86_64.rpm.zip
	   - tasks
	       + main.yml
	   - templates
	       + xe.rsp

#### Files
This is where we put the Zip archive that contains the Linux RPM for installing Oracle. 

#### Tasks
This is where we put the files describing the tasks to be performed as part of running this playbook. It’s the meat if you like. If you have a look at this main.yml task file you can see it’s fairly simple. Breaking it down we do the following:

- Install some required packages using yum
- Unzip the archive containing the RPM
- Install the RPM
- Create the response file used as part of the post install configuration
- Run the post install configuration
- Add the oracle env variables to the vagrant users bash profile 

#### Templates
This where we put our template files. These are files used in the installation that will have host specific values. For example in our case we’re creating a response file (xe.rsp) for the Oracle installer that provides values for the database ports and credentials.

### Installing Tomcat
The role we’ve created for installing tomcat is very similar in structure and is based on the ansible provided example here - [https://github.com/ansible/ansible-examples/tree/master/tomcat-standalone][9]

	- tomcat
	   - files
	       + tomcat-initscript.sh
	   - tasks
	       + main.yml
	   - templates
	       + server.xml
	       + tomcat-users.xml

#### Files
This includes our service script for tomcat. 

#### Handlers
This directory includes a file called main.yml that describes operations that are run upon notification - a good example, and the one used here is restarting a service when a configuration file changes.

#### Tasks
Again this main.yml file describes the tasks required to install and configure tomcat. They can be broadly described as:

- Install Java 
- Download the Tomcat binary
- Create install directories and SymLinks
- Create Tomcat users/groups
- Make sure all directories are owned by the new tomcat user
- Create the Tomcat server.xml configuration file
- Create the Tomcat tomcat-users.xml configuration file
- Put the Tomcat initscript in place

#### Templates
Our template files here are the server.xml and the tomcat-user.xml file

### Automating a Web Center Sites Installation
With our database and application server in place the next step is to perform that actual installation of Web Center Sites.

Let’s start by looking at our ansible role - this should all be starting to look familiar to you by now.

	- web-center-sites
	   - files
	       + ofm_sites_generic_11.1.1.8.0_disk1_1of1.zip
	       + ojdbc6.jar
	   - tasks
	       + main.yml
	   - templates
	       + catalina.properties
	       + catalina.sh
	       + create_wcs_user.sql
	       + customBeans.xml
	       + generic_omii.ini
	       + install.ini
	       + installer.sh
	       + server.xml

#### Files
The in this directory are some of the ones we downloaded previously namely the WCS installer archive and the Oracle JDBC driver.

#### Tasks
Our main.yml tasks file includes the following instructions:

- Install Java
- Install NetCat (used for running the installer in the background and to be able to pipe input)
- Create the database setup script
- Create the database
- Unzip the WCS installation archive
- Create directory for the installer to run from
- Unzip the Sites part of the install suite
- Create shared directory
- Create the WCS home directory
- Move the JDBC driver into the tomcat lib
- Add WCS parameters to the tomcat startup script (catalina.sh)
- Create the ini file that drives the silent installer
- Create the installer configuration file
- Update the tomcat server.xml file with details of the WCS datasource
- Update the catalina.properties file to add the WCSHOME/bin directory to the classpath
- Create the installer.sh silent install wrapper script
- Execute the installer.sh script
- Update the customBeans.xml configuration file that controls the urls that WCS trusts access from
- Restart tomcat\_ 
Which is a whole lot of stuff going on. Of all of it perhaps the most interesting is the installer.sh script (the rest is fairly standard installing WCS fare) so let’s continue by having a look at what that script is doing.


	#!/bin/bash
	
	echo "<<< deploying sites"
	cd {{wcs_installer_directory}}/Sites
	rm out.log
	touch out.log
	nc -l 12345 | sh csInstall.sh -silent | tee out.log &
	while ! tail -n 1 out.log | grep "press ENTER."
	do sleep 1 ; echo ...deploying...
	done
	echo ">>> deploying sites"
	
	echo "<<< starting tomcat"
	/usr/share/tomcat/bin/startup.sh
	while ! wget -q -O- http://{{wcs_install_webserver_address}}:{{wcs_install_webserver_port}}/cs/HelloCS | grep reason=Success
	do echo "...starting..." ; sleep 1
	done
	echo ">>> started tomcat"
	
	echo "<<< installing sites"
	cd {{wcs_installer_directory}}/Sites
	echo | nc localhost 12345
	while ! tail -n 1 out.log | grep "Installation Finished Successfully"
	do sleep 1 ; echo ...installing...
	done
	echo ">>> installed sites" 


Of interest here is the use of NetCat to provide a way of running the installer that allows us to send an ‘Enter’ command to the installer process once we’ve checked that the web application has deployed correctly.

### Introduction to Packer
So now we have our recipes for automating installation and configuration but we still have a few problems to solve. 
The first is that all of this takes a little while to complete. 
If we want our developers to be able to recreate their development environments as easily as possible, we also need them to be able to do so as quickly as possible and at the moment that’s not really the case. We also have a little hack in our vagrant file that I promised we would remove. We can solve both of these problems by using a tool called packer. 

I like to think of packer as a golden master producer, it gives you the ability to “create machine and container images for multiple platforms from a single source of configuration.”

In practice this means we can use packer to create our development environments as self contained vagrant boxes (as in boxes with all the provisioning already completed). Which is fine although it’s worth noting that we could just do this with Vagrant itself. However it’s real value comes with being able to build for multiple platforms and containers - imagine being able to take our development environment and create it as an ami for running on Amazon Web Services or as a VMWare VM for deploying into VSphere or even as a docker container. Imagine being able to create all of our environments from dev all the way through to production from the same version controlled configuration! Exciting stuff indeed.

In the mean time it also means that we can fix our issue with swap space and the oracle xe installer when we create the our initial virtual machine and as a side effect remove our dependency on the bento virtual box.

First off we’ll start by looking at how packer works and once we’ve configured the build of our VM we’ll look at adding some ansible provisioning to automate the install of Web Center Sites and all it’s associated software.

You can have a look at our sample packer project by cloning it from here

	git clone https://github.com/Manifesto-Digital/wcs-env-tutorial-packer-templates.git

#### Layout of packer project

	+ wcs-centos-6.6-x86_64.json
	- ansible
	  + hosts
	  + wcs.yml
	  - group_vars
	  - roles
	- http
	  - centos-6.6
	    + ks.cfg
	- scripts
	  - centos
	    + cleanup.sh
	    + fix-slow-dns.sh
	  - common
	    + ansible.sh
	    + minimize.sh
	    + shutdown.sh
	    + sshd.sh
	    + sudoers
	    + vagrant.sh
	    + vmtools.sh
	- vagrantfile_templates
	  + macosx.rb

The most important file in this list is wcs-centos-6.6-x86_64.json.
It’s the main packer configuration file and describes the how and what that we’re going to build. Let’s have a look in a little more detail.

At the top level we have the following objects

- builders
- post-processors
- provisioners
- variables

#### Builders
A builder in packer is responsible for creating the machines and generating images from them for the platforms we’re targeting (in our case that’s just VirtualBox for now)

#### Provisioners
This section contains the list of all the provisioners that will be run as part of the creation of a machine image. In our case we’re going to plugin the ansible playbook and roles that we’ve already developed. 

#### Post Processors
Post processors describe any tasks that need to be performed on machine images once the building and provisioning is complete. In our scenario we’re only using one that creates us our vagrant box - however this would be the place to configure things like pushing a docker build to a registry.

#### Variables
Variables holds any user defined variables we need to use as part of our build.

### Packer, Provisioners and Ansible
Before we run our packer build we should talk a little bit about what we need to do to get our ansible playbook running as a provisioner in packer.

The following section from our configuration is the pertinent part, it describes two things, firstly installing ansible in our   local environment and secondly running the packer provided ansible provisioner. 

	   {
	      "type": "shell",
	      "execute_command": "echo 'vagrant' | {{.Vars}} sudo -S -E bash '{{.Path}}'",
	      "script": "scripts/common/ansible.sh"
	    },
	    {
	      "type": "ansible-local",
	      "playbook_file": "ansible/wcs.yml",
	      "group_vars": "ansible/group_vars",
	      "inventory_file": "ansible/hosts",
	      "role_paths": [
	        "ansible/roles/common",
	        "ansible/roles/oracle",
	        "ansible/roles/tomcat",
	        "ansible/roles/web-center-sites"                
	      ]
	    },  

In order to install ansible we’ve written a short shell script that looks like this

	#!/bin/bash -eux
	
	# Install EPEL repository.
	rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
	
	# Install Ansible.
	yum -y install ansible

We then pass some configuration to the ansible provisioner about where our roles are in our packer project filesystem, where our inventory file is, where our group variables are and finally where our main playbook file is. The playbooks are exactly the same as the ones we used when we’re working with Vagrant directly.

Now is the time (finally) to run packer - assuming of course that you haven’t already ;)

	packer build wcs-centos-6.6-x86_64.json

You should see a whole series of things start to happen on your machine now - packer will start by downloading the Centos installer DVD iso (this can take a while), then it will run an unattended installation and finally provision the image and then create a Vagrant box. Once all of this is finished we can add the newly minted Vagrant box to Vagrant using this command

	vagrant box add wcs-centos-6.6 builds/wcs-centos-6.6.virtualbox.box

### Testing our new Vagrant Box

Now to test we can update our Vagrantfile to look a bit more like this:

	# -*- mode: ruby -*-
	# vi: set ft=ruby :
	
	# All Vagrant configuration is done below. The "2" in Vagrant.configure
	# configures the configuration version (we support older styles for
	# backwards compatibility). Please don't change it unless you know what
	# you're doing.
	
	Vagrant.configure(2) do |config|
	
	  config.vm.box = "wcs-centos-6.6"
	
	  # Application server.
	  config.vm.define "wcs-server" do |app|
	    app.vm.hostname = "wcs.192.168.60.5.xip.io"
	    app.vm.network :private_network, ip: "192.168.60.5"
	    app.vm.network "forwarded_port", guest: 8080, host: 8080, auto_correct: true
	    app.vm.network "forwarded_port", guest: 1521, host: 1521, auto_correct: true      
	    app.vm.network "forwarded_port", guest: 9090, host: 9090, auto_correct: true
	  end  
	end
	

This Vagrantfile is now using our newly created box and although we’re still specifying our networking we’ve removed all of the provisioning as it’s not needed. To start it all up run…

	vagrant up

… and you should be able to browse to [http://wcs.192.168.60.5.xip.io:9090/cs/][10] and log in to WCS using the standard fwadmin credentials.


### Next steps
Now that you’ve automated the creation of your WCS development environments there are a number of directions you could pursue to take things further.

#### Making the Vagrant Boxes Accessible
Vagrant can download boxes from Urls so perhaps make it easier for colleagues to get hold of box updates by placing your box on a web server for easier distribution.  

#### Creating other types of machine images
One of the big benefits of packer is the ability to create multiple machine image formats from one configuration file - how about creating a docker image for your WCS development environments as well?

Let me know how you get on.


[1]:	https://www.virtualbox.org/wiki/Downloads
[2]:	http://www.vagrantup.com/downloads
[3]:	https://www.packer.io/intro/getting-started/setup.html
[4]:	http://www.oracle.com/technetwork/middleware/webcenter/sites/downloads/index.html
[5]:	http://www.oracle.com/technetwork/database/database-technologies/express-edition/downloads/index.html
[6]:	http://www.oracle.com/technetwork/apps-tech/jdbc-112010-090769.html
[7]:	https://github.com/chef/bento
[9]:	https://github.com/ansible/ansible-examples/tree/master/tomcat-standalone
[10]:	http://wcs.192.168.60.5.xip.io:9090/cs/
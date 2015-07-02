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

### Introduction to Ansible
TBD
### Installing Oracle XE
TBD
### Installing Tomcat
TBD
### Automating a Web Center Sites Installation
TBD
### Introduction to Packer
TBD
### Packer, Provisioners and Ansible
TBD

### Making the Vagrant Boxes accessible
TBD

[1]:	https://www.virtualbox.org/wiki/Downloads
[2]:	http://www.vagrantup.com/downloads
[3]:	https://www.packer.io/intro/getting-started/setup.html
[4]:	http://www.oracle.com/technetwork/middleware/webcenter/sites/downloads/index.html
[5]:	http://www.oracle.com/technetwork/database/database-technologies/express-edition/downloads/index.html
[6]:	http://www.oracle.com/technetwork/apps-tech/jdbc-112010-090769.html
[7]:	https://github.com/chef/bento
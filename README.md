# Part 1: Creating Web Center Sites Environments
For this, the first tutorial in our Continuous Delivery series, we’re starting right at the beginning with a look at creating environments to support Web Center Sites development.

We’ll look at creating environments that are as alike as we can make them, up through the stack - from development through test up to production environments. We’ll also look at automating this process because AUTOMATE ALL THE THINGS.

In more detail we’ll look first at creating a Web Center Sites development environment. To do that we’ll need to automate the installation and configuration of Web Center Sites. In order to install WCS we also need to install and configure some supporting software. 

This includes:
- An operating system (Centos 6.6)
- An application server (Tomcat 7.0.62)
- A database (Oracle 11g XE)

To make all of that repeatable, we’ll be encapsulating all the logic by using a Configuration Management tool called Ansible.

Finally we’ll use a tool called packer.io to create machine images that make it quick and easy for our developers to get a development environment up and running.

Before we get going though, it’s probably important to talk a little about why we would want to do any of these things. I mean this setup takes a little bit of work so it’s good to start with a business case of sorts.

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
TBD
### Introduction to Vagrant
TBD
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


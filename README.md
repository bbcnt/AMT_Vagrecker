AMT_Vagrecker
=============
## B. Carvalho Bruno & Bignens Julien  ##

This is the repo for the project number 2 of AMT, 2014-2015. You will find here everything you need to get started with our implementation.

# Quick Overview #

Let's see how the architecture is built and how it works.

![Schema](./images/schema_fonctionnement.png "First Aspect")

The client can access the web application by accessing the HAProxy container, which is a load balancer proxy server. This server will "serve" one of the three availables GlassFish web applications that are running simultaneously.

Meanwhile, a developper can access Jenkins and PHPMyAdmin, to be able to work on the CI (Continuous Integration) of the project. The Glassfish instances and the PHPMyAdmin (PMA) containers also interract wtih MySQL, which is a MySQL server. Since there is only one Database, any GF server can access the same data, which is **normal**. Jenkins can communicate with GitHub, to be able to pull the most recent source code and deploy it automatically on Glassfish.

This is the basic implementation, let's see precisely how it works now.

# From a Maven project to a web application #

Let's try to decorticate the [MO](http://en.wikipedia.org/wiki/Modus_operandi) of the our implementation. So first, what do we have ?

- The source code of the [first project](https://github.com/bbcnt/AMT_REST) 
- And that's all really...

We have a maven project and we want to deploy it on a glassfish instance. The idea here is :

1. Generate a .war file from the Maven project with Jenkins (the Maven project is pulled from GitHub)
2. Deploy this project on a glassfish instance
3. Link everything so the glassfish server can work with the MySQL database.
4. Add a few tools, like PHPMyAdmin, to be able to work efficiently with MySQL.
5. Add a few other instances of Glassfish (replicates of the first one).
6. Put in some load balancing and we are done.

So we do have a few steps ahead of us...

First, we are going to talk about the linked containers and shared volumes.

# Shared volumes and links #

For containers to be able to "talk to each other", we need to create *links* between them. Also, to be able to share data, we need to create volumes. There are two kinds of shared volumes, we can either create a container that will only serve as a data container, or we can directly share a directory from an existing container. In this lab, we only use the second option, since we pretty much do everything with scripts and ADD (using Dockerfiles) the files we need. 

So, this is the shared volumes structure of our architecture:

![Schema](./images/schema_volumes.png "First Aspect")

Basically, pretty much all containers are linked to the mysql one, and we have a share in the server gf1 (we shall explain why later). Also, for haproxy to be able to work with the glassfish servers, it needs to be linked to them. Jenkins only works with gf1, again, this is used with the shared volume, we'll talk about this later.

In the Vagrantfile, we can see these operations here, for example: 

	d.run "phpmyadmin", image: "heig/phpmyadmin", args: "-p 5050:80 --link mysql:mysql"
	OR
	d.run "gf1", image: "heig/glassfish", args: "-p 4081:4848 -p 4082:8080 --link mysql:mysql -v /opt/glassfish/glassfish4/glassfish/domains/domain1/autodeploy/"

As promised, we will now see why we share the autodeploy directory of gf1.

## Some (dirty) magic ##

Since we have every glassfish instances that come from the same image, they all are the same. Also, there is a pretty cool feature in GF. If you put a .war file inside the autodeploy directory of your domain (here it's the default domain1), it will be automatically deployed on Glassfish (no need to use asadmin or else). So, why is this cool? 

Well, simple, since every glassfish instance is the same, they have the same structure, and of course, the autodeploy directory is in the same place for each of them. So, let's say we only place the .war file in gf1's autodeploy dir, and all the other gf instances map their own autodeploy dir with this one, putting the file on gf1 will put it on all the gf instances. So it's really easy to make this from Jenkins to gf1 (this is why Jenkins is only linked with gf1).

We create the .war file, place it on the autodeploy dir of gf1 (using the shared volume), all the other gf instances synchronize their content with gf1, so they all get the .war file and deploy it automatically. If this is not magical, I don't know what is.

So instead of having some complicated script in the post-treatment of Jenkins, we only have 

	cp $JENKINS_HOME/jobs/AMT_REST/workspace/target/AMT_REST-1.0-SNAPSHOT.war /opt/glassfish/glassfish4/glassfish/domains/domain1/autodeploy/AMT_REST.war

A stupid cp command that will take the created war, rename it in a more friendly name, copy it inside the shared volume (note that you access shared volumes by the same path, so it's /opt/glassfish/glassfish4... which is the path on the gf1 instance). And after that, the deployment is done. A flag file can be found when the autodeploy has succeeded.

# Running the test implementation #

Ok, so we have seen how the architecture works somehow (we'll discuss each container later) and we now know what it does. So let's now see how to use it.

First of all, everything has been automated, so the only thing you'll have to do is... *press a button*. The automatisation works with our project 1, called AMT_REST. The documentation for the project itself is [here](https://github.com/bbcnt/AMT_REST). Now, let's do this.

### First point ###

Clone [this repo](https://github.com/bbcnt/AMT_Vagrecker). It contains all the Dockerfiles to create the architecture. 

### Second point ###

Get inside the clones repo and at the root (where the vagrantfile is), just type :

	vagrant up

Let is be known that the process is quite lenghty in time (one hour at least). The good point is, you don't have to do anything so you can enjoy a good coffee or else ;)

You may need to provision the VM (not needed on the first start), to do so :

	vagrant provision

Now sit back and relax...

### Third point ###

Now, the last point, the only thing you need to do is, get here :

	http://localhost:9001

This is the access to the jenkins application. Remember when we told you you would have to press a button? This is it.

![Jenkins_build](./images/jenkins_build.png "First Aspect")

And... that's all. You can now access :

	http://localhost:9002

And you have the deployed application (if you get error 404, the deploy can take a few minutes).

We are done with the example, the next parts will be about the implementation.

# Port Mapping #

Before we get to the containers, let's see the port mapping between the Host, the vagrant VM and the Dockers. As described in the Vagrantfile :

    #These are the port forwarding that are configured
	config.vm.network "forwarded_port", guest: 7070, host: 9001 # Jenkins
	config.vm.network "forwarded_port", guest: 9090, host: 9002 # Reverse proxy (entrypoint for users)
	config.vm.network "forwarded_port", guest: 3306, host: 9003 # MySQL Server (pretty useless as is)
	config.vm.network "forwarded_port", guest: 5050, host: 9004 # PHPMyAdmin (use localhost:9004/phpmyadmin)
	config.vm.network "forwarded_port", guest: 4081, host: 9005 # GF1 config (4848)
    config.vm.network "forwarded_port", guest: 4082, host: 9006 # GF1 web (8080)
    config.vm.network "forwarded_port", guest: 4084, host: 9007 # GF2 web (8080)
    config.vm.network "forwarded_port", guest: 4086, host: 9008 # GF3 web (8080)


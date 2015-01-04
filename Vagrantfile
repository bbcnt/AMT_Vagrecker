# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile created by Brito Carvalho Bruno for AMT Project on Vagrant / Docker architecture
# All documentations on : https://github.com/bbcnt/AMT_Vagrecker
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

    config.vm.box = "phusion-open-ubuntu-14.04-amd64"
    config.vm.box_url = "https://oss-binaries.phusionpassenger.com/vagrant/boxes/latest/ubuntu-14.04-amd64-vbox.box"


	# Create a private network, which allows host-only access to the machine
	# using a specific IP.
	config.vm.network :private_network, ip: "192.168.33.20"

    #These are the port forwarding that are configured
	config.vm.network "forwarded_port", guest: 7070, host: 9001 # Jenkins
	config.vm.network "forwarded_port", guest: 9090, host: 9002 # Reverse proxy (entrypoint for users)
	config.vm.network "forwarded_port", guest: 3306, host: 9003 # MySQL Server (pretty useless as is)
	config.vm.network "forwarded_port", guest: 5050, host: 9004 # PHPMyAdmin (use localhost:9004/phpmyadmin)
	config.vm.network "forwarded_port", guest: 4081, host: 9005 # GF1 config (4848)
    config.vm.network "forwarded_port", guest: 4082, host: 9006 # GF1 web (8080)
    config.vm.network "forwarded_port", guest: 4084, host: 9007 # GF2 web (8080)
    config.vm.network "forwarded_port", guest: 4086, host: 9008 # GF3 web (8080)
    config.vm.network "forwarded_port", guest: 4087, host: 9009 # HAProxy (stats on localhost:9009/haproxy login: admin, admin)
	

    # ==============================================================================
    # Provisioning, part 1: we run a script to reset the Docker environment
    # ==============================================================================
	
    #Deleting all the dockers (because it gets heavy as hell at some point if we don't)
    $script = <<SCRIPT
    
	echo Cleaning up Docker containers...
    sudo service docker restart
    sleep 3
	docker stop jenkins || true
	docker rm jenkins || true
	docker stop gf1 || true
	docker rm gf1 || true
	docker stop gf2 || true
	docker rm gf2 || true
	docker stop gf3 || true
	docker rm gf3 || true
	docker stop mysql || true
	docker rm mysql || true
	docker stop phpmyadmin || true
	docker rm phpmyadmin || true
    docker stop haproxy || true
    docker rm haproxy || true
	
SCRIPT
	
	config.vm.provision "shell", inline: $script
	
    # ==============================================================================
    # Provisioning, part 2: we use Docker to package and run services in our VM
    # ==============================================================================

	config.vm.provision "docker" do |d|

        #MySQL Server
		d.build_image "/vagrant/docker/mysql", args: "-t heig/mysql"
        d.run "mysql", image: "heig/mysql", args: "-p 3306:3306"
        
        #PHPMyAdmin (works with the MySQL server with the link)
		d.build_image "/vagrant/docker/phpmyadmin", args: "-t heig/phpmyadmin"
		d.run "phpmyadmin", image: "heig/phpmyadmin", args: "-p 5050:80 --link mysql:mysql"

        #Glassfish Servers
        d.build_image "/vagrant/docker/glassfish", args: "-t heig/glassfish"
        
        #Gf1 is sharing the autodeploy repertory, once a war file is put in this directory, it is automatically deployed by GF.
        d.run "gf1", image: "heig/glassfish", args: "-p 4081:4848 -p 4082:8080 --link mysql:mysql -v /opt/glassfish/glassfish4/glassfish/domains/domain1/autodeploy/"
        
        #And here, we use link the autoeploy repo of gf1, meaning, all the other GF servers will be updated once we update the first one.
		d.run "gf2", image: "heig/glassfish", args: "-p 4083:4848 -p 4084:8080 --link mysql:mysql --volumes-from gf1"
		d.run "gf3", image: "heig/glassfish", args: "-p 4085:4848 -p 4086:8080 --link mysql:mysql --volumes-from gf1"

        #Jenkins Server
        
        #We create out .war file from the project, then we put it directly into gf1 autodeploy directory. We don't need to use 
        #asadmin or anything else, it will be deployed automatically
        d.build_image "/vagrant/docker/jenkins", args: "-t heig/jenkins"
		d.run "jenkins", image: "heig/jenkins", args: "-p 7070:8080 --link gf1:gf1 --volumes-from gf1"
        
        #Reverse Proxy (technically, the user only needs to connect to localhost:9002 and he will access one of the 3 GF instances)
        
        d.build_image "/vagrant/docker/haproxy", args: "-t heig/haproxy"
		d.run "haproxy", image: "heig/haproxy", args: "-p 9090:80 -p 4087:8080 --link gf1:gf1 --link gf2:gf2 --link gf3:gf3"
        
        
	end
	
end

# Running Order Service in a Docker Container #
## Overview ##
This repository contains an example application (OrderService), represented as a compiled Java Solution packaged as a Jar file. This application represents the MRP portion of the overall solution.

To run this application, this set of Docker files and scripts runs two different containers:
 
1. Mongo DB
1. Order Service

## Running Solution ##

The execution for this example application is broken down into 2 steps and contained within 2 shell scripts:

1. buildboth.sh
1. runboth.sh

### Build ###

By executing the shell script 'buildboth.sh' - this will build the Docker container and create several Docker images within the Docker host (note that there are dependencies that are automatically pulled to the Docker host).

When done you can list the 'images' for Docker using 

    docker images

This will output a list of images similar to the following:

	REPOSITORY              TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
	scicoria/orderservice   0.1                 416cc921bc78        14 minutes ago      501.9 MB
	scicoria/mongoseed      0.1                 785401816c4d        16 minutes ago      500.8 MB
	ubuntu                  14.04               2d24f826cb16        12 days ago         188.3 MB
	ubuntu                  14.04.2             2d24f826cb16        12 days ago         188.3 MB
	ubuntu                  latest              2d24f826cb16        12 days ago         188.3 MB
	ubuntu                  trusty              2d24f826cb16        12 days ago         188.3 MB
	ubuntu                  trusty-20150218.1   2d24f826cb16        12 days ago         188.3 MB
    java                    8-jre               b0f21df5333b        5 weeks ago         478.7 MB


Note that there are several Ubuntu images. Docker provides the isolation among the containers allowing different packages to leverage different versions of dependent packages.

---
### Run ###
Once you've created the Docker images they can be executed. To run, execute the 2nd script - runboth.sh.

Once you've run the script, Docker client will respond with 2 Container ID's.  To list the running containers, execute the following:

    docker ps

This will output something similar to the following.

	CONTAINER ID        IMAGE                       COMMAND                CREATED             STATUS              PORTS                      NAMES
	3c18e9f84637        scicoria/orderservice:0.1   "java -jar ordering-   14 minutes ago      Up 14 minutes       0.0.0.0:8080->8080/tcp     dreamy_torvalds
    a3ac3c44140d        scicoria/mongoseed:0.1      "./mongorun.sh"        14 minutes ago      Up 14 minutes       0.0.0.0:27017->27017/tcp   mongodb           

Make note of the 'Names' column. Note that the '/mongoseed' (running Mongod DB) container is named 'mongodb'.  While the other container is assigned a random, but interesting name - here it just happens to be 'dreamy_torvalds'.  Yours will be different.

Naming the container is important when you need to 'connect' between containers - this is 'linking' and this allows the 'OrderService' container to talk to the 'mongoddb' named container.   As you may see if you continue with docker, naming the container makes management via the Docker CLI easier instead of using the long 'CONTAINER ID' generated by Docker daemon.

---
### Quick Test ###
There's a couple things you can verify to see if all is OK

- Check the /data/db directory
- Run curl against 'Order Service'

#### Mongo DB Data Directory ####
Make note of where the Mongo DB data directory has been created in this example.  By running

    ls -l /data/db

You should see the Mongo DB data files, and 2 with a first name of 'ordering' - these files represent the seed data that was inserted as part of the Docker 'run' - take a look at the ./mongoseed/Dockerfile for more information.

	drwxr-xr-x 2 root root     4096 Mar  6 00:41 journal
	-rw------- 1 root root 16777216 Mar  6 00:41 local.0
	-rw------- 1 root root 16777216 Mar  6 00:41 local.ns
	-rw-r--r-- 1 root root     4794 Mar  6 00:41 mongodb.log
	-rwxr-xr-x 1 root root        3 Mar  6 00:40 mongod.lock
	-rw------- 1 root root 16777216 Mar  6 00:41 ordering.0
	-rw------- 1 root root 16777216 Mar  6 00:41 ordering.ns
    drwxr-xr-x 2 root root     4096 Mar  6 00:41 _tmp

#### Running Curl to validate ####
At this point, as you see in the prior step above, you can issue an HTTP GET to the listener process for OrderService, which is listening on port 8080 locally - and has been exposed from the Docker container by way of the '-p' Docker CLI switch.

	curl http://localhost:8080/catalog


This should return a json payload as follows:

	[
	    {
	        "skuNumber": "ACC-0001",
	        "description": "Shelving",
	        "unit": "meters",
	        "unitPrice": 10.75
	    },
	    {
	        "skuNumber": "ACC-0002",
	        "description": "Refrigeration Unit",
	        "unit": "",
	        "unitPrice": 1500.0
	    },
	    {
	        "skuNumber": "ACC-0003",
	        "description": "Freezer Unit",
	        "unit": "",
	        "unitPrice": 4500.0
	    },
	    {
	        "skuNumber": "ACC-0004",
	        "description": "Fancy Shelving",
	        "unit": "meters",
	        "unitPrice": 30.5
	    }
	]





## Mongo DB ##

This example makes use of a Docker file that is based off of ubuntu:latest.  The docker file is below.

Note that this runs a script that starts Mongo Daemon - waits for a ready state by polling a file, then upon the start inserts seed data - which is present in the directory and part of the Docker build for that package.


		# Dockerizing MongoDB: Dockerfile for building MongoDB images
		# Based on ubuntu:latest, installs MongoDB following the instructions from:
		# http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/
		
		FROM       ubuntu:latest
		MAINTAINER Shawn Cicoria shawn.cicoria@microsoft.com
		
		ADD mongorun.sh .
		# Installation:
		# Import MongoDB public GPG key AND create a MongoDB list file
		RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
		RUN echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | tee /etc/apt/sources.list.d/10gen.list
		
		# Update apt-get sources AND install MongoDB
		RUN apt-get update && apt-get install -y mongodb-org
		
		# Create the MongoDB data directory
		RUN mkdir -p -m 777 /data/db
		
		RUN mkdir -p -m 777 /tmp
		
		COPY mongodb.catalog.json /tmp/mongodb.catalog.json
		COPY mongodb.dealers.json /tmp/mongodb.dealers.json
		COPY mongodb.quotes.json /tmp/mongodb.quotes.json
		COPY mongorun.sh /tmp/mongorun.sh
		
		WORKDIR /tmp
		
		RUN chmod +x ./mongorun.sh
		
		# Expose port #27017 from the container to the host
		EXPOSE 27017
		
		VOLUME /data/db
		
		ENTRYPOINT ["./mongorun.sh"]

## Order Service ##

The order service runs a Java process - and that process creates the HTTP listener on port 8080.


	FROM java:8-jre
	MAINTAINER Shawn Cicoria shawn.cicoria@microsoft.com
	
	
	ENV APP_HOME /usr/local/app
	ENV PATH $APP_HOME:$PATH
	RUN mkdir -p "$APP_HOME"
	
	WORKDIR $APP_HOME
	
	ADD ordering-service-0.1.0.jar $APP_HOME/
	
	EXPOSE 8080
	CMD ["java", "-jar", "ordering-service-0.1.0.jar"]




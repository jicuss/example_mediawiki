# example_mediawiki

## What is this?
This is a collection of scripts that will instantiate a basic mediawiki install using
* mediawiki [1.29.1] running on PHP 7.1.11 through Apache
* mariadb [10.2.10] - The open source fork of mysql
* jenkins [2.89] - a common build tool, used to schedule backups, coordinate deployments
* adminer [v XXX] - [not required] a super basic db exploration tool tool to test db connectivity / queries

## What is Docker and why should I use it?
Docker is a lightweight way of describing the environmental conditions required to run arbitrary code isolated from your host. Many people run their servers inside of a VM. This allows them to isolate their services to avoid dependency conflicts, and sandbox in hackers away from their primary host.

Docker is essentially a VM. To run a VM you would allocate specific resources to it (CPU / memory) and those resources would be unavailable to the other VMs. Docker aims to allow you to run a greater number of services, and start them more quickly by will sharing a pool of resources.

It accomplishes this goal by replacing a component of the VM architecture called the hypervisor. This component is the piece responsible for translating the low level processor language to another architecture. Its why you can run a linux, windows, or OSX VM on any machine. The thing is that its relatively expensive as you need to run a completely new copy of the OS for every container. Docker replaced that component with a driver that emulates the kernel for the environment by passing arguments to the host machines kernel. That allows you to run essentially one OS and apply a layer on top of that. Those code layers can run whatever 'OS' you want in isolation, start quickly, and share resources

## How does that help with deployment?
A real world example illustrates this. Lets take Jenkins.  The day before I put this together, a security vulnerability notice was posted for the version of Jenkins I was using (2.84). Docker made it extremely easy to upgrade to a safe version. The jenkins core team manages a docker image that I used as the base of my own image. I pulled down the latest jenkins image, rebuilt my image, and restarted the container. voila! an up to date version was running.

```
# =========================================================
#          pull and rebuild single image
# =========================================================

docker pull jenkins/jenkins
docker-compose build jenkins

# =========================================================
#          update and rebuild all images at once
# =========================================================

docker-compose pull
docker-compose build
docker-compose up -d
```

But what if that didn't work? Easy, just roll back to a known working copy. Docker uses a concept of tags to label a specific version of an image.
Since we are applying additional layers to jenkins, you would modify the FROM tag at the beginning of the file. Something like:
> FROM jenkins/jenkins:2.84

If you wanted to use a specific version of an image you are not building, specify the image version in the compose file. I forgot to tag that version, so I passed the hash as a tag in this example
>  jenkins:

>    image: jenkins/jenkins:6086cb8a9609

A common thing would be to tag production and experimental versions separately. That way you could run two completely different versions of the application and upgrade once you feel comfortable by simply changing the compose file and restarting

* https://jenkins.io/security/advisory/2017-11-08/

## How do I install Docker?
* windows. never used, there is a GUI installer
* OSX. download the GUI installer
* Linux. Install the base docker package, along with docker-compose.
```
    yum install docker-ce
    pip install docker-compose
```

## How do I run this?
* first you'll need to build the containers. This would happen automatically if you tried to start the container and it was never built. Anytime you change the dockerfile, you'll need to rebuild the image.
```
    docker-compose -f docker-compose.yml build
```

* run the docker-compose file, this will start the services in the background (daemon mode)
```
    docker-compose -f docker-compose.yml up -d
```

* you will need to instantiate the DB the first time you run it
```
    docker-compose -f docker-compose.yml run jenkins /app/bootstrap.sh
```


## What's included here?
* a very basic install of mediawiki. Minimal changes to LocalSettings.php. I installed plantuml, an extension I like for flow diagrams.
* there are two copies of the mediawiki server and backend DB. A 'prod' version and a 'QA' version.
* a jenkins job to backup the prod DB to a SQL file nightly.

## URLS:
* Jenkins - http://localhost:8080
* Mediawiki Prod - http://localhost:80
* Mediawiki QA - http://localhost:85

## Basic Docker Commands
```
# =========================================================
#                        general docker
# =========================================================

# delete all images
docker rmi $(docker images -q) -f

# delete all containers
docker rm $(docker ps -a -q) -f

# delete all volumes
docker volume rm $(docker volume list) -f


# =========================================================
#                working with single containers
# =========================================================

# build an image from current folder (in this case named postgres)
docker build -t postgres .

# list all active containers (gives name and ID)
docker ps

[vagrant@local-development ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                      NAMES
65045bc2b817        docker_vertica      "/docker-entrypoin..."   9 hours ago         Up 33 seconds       0.0.0.0:5433->5433/tcp     vertica


# =========================================================
#                various ways of opening a terminal into a container
# =========================================================

# open a terminal by container ID (container is running)
docker exec -i -u root -t ecfbd489c022 /bin/bash
docker exec -i -u jenkins -t 06d3a8acc06b /bin/bash
docker exec -i -u airflow -t f95dad9226fe /bin/bash

# Run an interactive terminal in a non running container
docker run -it bbc61e159f18 /bin/bash

# open a terminal by container name
docker exec -i -t vertica /bin/bash

# run container from non standard compose file (and include servie ports)
docker-compose -f docker-compose-servers-splunk.yml run --service-ports splunk

# kill all running containers
docker kill $(docker ps -aq)
docker service rm $(docker service ls -q)

# copy a file from a running container
docker cp some-nginx:/etc/nginx/nginx.conf /some/nginx.conf

# =========================================================
#                   docker compose specific
# =========================================================

# build several containers defined in a compose file
docker-compose build

# start compose in daemon mode (in the background)
docker-compose up -d

```

## Things not included with this
#### A reverse proxy
* you'll want to spin up a reverse proxy to tunnel incoming traffic from a public network through to the servers using an SSL certificate. That will enable HTTPS support, preventing passwords from being passed in plaintext over the network
* nginx is a good choice, you should place mediawiki and jenkins behind it

#### a public and private namespace
* docker supports multiple networks. You should ideally have a public and a private one. The reverse proxy would be in the public one, with the db, mediawiki, and jenkins in the private one.

#### Customization of mediawiki.
* I used a vanilla build, only installed plantuml. There are some cool extensions out there, you may want to install more.

## Known Issues With Docker
#### Linux
* **the bridge network**. no known issues. This compose file will use the bridge adapter exclusively as it will be run on a single box and doesn't need the routing features offered by the overlay.

* the **overlay networking driver** provides an easy to use mechanism to route traffic to arbitrary containers based on properties such as their name, tags, functionality, or physical location. Mostly works well.
    * allows you to run multiple copies of a service in parallel behind an automatically configured load balancer. if one goes down, you're still good and the service will perform to the user normally
    * when a container is unreachable or crashes, traffic will still route to a container running that service
    * Be aware there is a known issue with individual containers becoming unresponsive through the overlay load balancer if they crash in a weird way. Seems like an edge case, I've only seen it happen once. Fix by shutting the containers down, remove containers and networks, restart. This will accomplish that.

```
# =========================================================
#           a script to completely wipe and restart a set of docker services
# =========================================================

echo "[CAUTION] THIS IS A VERY DRASTIC OPERATION!!!"
echo "This will eliminate all running containers and remove their associated networks:"
read -p "Are you sure? [yes/NO] " -n 1 -r
if [[ ! $REPLY =~ ^[Yy]$ ]]
then
    exit 1
fi


docker kill $(docker ps -aq)
docker container prune --force
docker service rm $(docker service ls -q)
docker network prune --force
```


### OSX
* the volume mount and image storage mechanism has a long known bug that causes the filesize of the docker storage to increase indefinitely. Wiping the docker storage and restarting the services works, but be mindful that you will need to backup your volumes first.

    * more information and fix script available @ https://blog.mrtrustor.net/post/clean-docker-for-mac/

### Windows
* haven't personally used, have read that windows 10 offers great docker support. the community seems less in agreement about windows 7

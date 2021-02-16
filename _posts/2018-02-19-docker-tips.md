---
layout: post
title: Docker Tips
date: 2018-02-19
categories: linux docker
---

Basic docker commands and tips.


## Run docker with ports and volumes

	docker run --name=elk -p 5601:5601 -p 9200:9200 -p 5000:5000 -v /Users/jaci/tmp/elk-data/:/var/lib/elasticsearch -it myelk:v0

	docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 -it --name elk sebp/elk

## Login to running container

	docker exec -t 7cb0aff2732b sh

```
docker ps
docker stats
docker search
docker search centos
docker pull centos:centos6
docker run -i -t centos /bin/bash
docker images
docker cp /home/shared/develenv-31.sh 379028b15a63:/develenv-31.sh
docker commit -m "added develenv.sh" -a "jaci" 379028b15a63 ci-mobtel/develenv:v1
docker run -i -t ci-mobtel/develenv:v1 /bin/bash
docker run -i -t ci-mobtel/develenv:v1 /develenv-31.sh
docker build -t ci-mobtel/develenv:v2 .
docker tag 7466d247fc8f test/develenv:latest
```

## Login to docker hub

	docker login $username


## Docker JOB variable
 
```
JOB=$(docker run -d ubuntu /bin/bash -c "while true; do echo ello debian; sleep 1; done")

docker stop $JOB
docker start $JOB
docker restart $JOB
docker kill $JOB
```

### Stop & Remove JOB

```
docker stop $JOB
docker rm $JOB
```


## UPDATE,CREATE, COMMIT docker images

```
docker pull $docker-image
docker run -i -t $dockerrepo/image /bin/bash
```

Update image as you whant and commit your changes

```
docker commit -m "added something" -a "newname" \$ImageID $dockerrepo/image:newversion
```

Push your changes 

```
docker push dockeruser/$docker-image:newversion
```

Remove the image

```
docker rmi dockeruser/$docker-image:newversion
```

## NETWORK PORT MAPPING

### arbitrary port

	docker run -d -P image/app python app.py

### fixed port

	docker run -d -p 80:5000 image/app python app.py


## CONNECT CONTAINERS

Linking container between them

```
$ docker run -d --name dblink training/postgres 
$ docker run -d -P --name web --link dblink:dblink training/webapp python app.py
```

Checkout if it works through name web

## ENVIRONMENT VARIABLES

```
docker run --rm --name web3 --link dblink:dblink training/webapp env
docker run -t -i --rm -link dblink:web training/webapp /bin/bash
```

## DATA VOLUMES

```
docker run -it --name container-data -h CONTAINER -v /data debian /bin/bash
```

```
$ docker inspect container-data
        "Mounts": [
            {
                "Name": "7166ebad3038b416977ca8d566d9b7598a23af831ed7607c456f371bc264c46d",
                "Source": "/var/lib/docker/volumes/7166ebad3038b416977ca8d566d9b7598a23af831ed7607c456f371bc264c46d/_data",
                "Destination": "/data",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
```

### MOUNT DATA CONTAINER FROM OTHER CONTAINER


	docker run -it -h NEWCONTAINER --volumes-from container-data debian /bin/bas

### BACKUP A DATA VOLUME

#### create volume

	docker create -v /testdb2 --name testdb2 training/postgres /bin/true 

#### backup volume

	docker run --volumes-from testdb2 -v $(pwd):/backup ubuntu tar cvf /tmp/testdb2.tar /testdb2


## Mixing options

```
$ docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 -p 5045:5045 -p 5046:5046 -p 9600:9600 -v /Users/jaci/tmp/elk-data/:/var/lib/elasticsearch -it myelk:ml1g
docker run -p 5601:5601 -v /var/log/nginx:/logs/nginx -v /var/log/op_test:/logs/op_test -it myelk:0
```

## REMOVE and update DOCKER & docker-compose

First remove any docker and/or compose installed on your host.

	$ apt-get remove $(dpkg -l |grep docker | awk {'print $2'})

After that, follow these instructions.

```
$ apt-get -y install apt-transport-https ca-certificates curl software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
$ apt-get update
$ dpkg -l |grep compose
$ apt-get remove docker-compose
$ apt-get install docker-ce
$ curl -L https://github.com/docker/compose/releases/download/1.19.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```
## Openvpn with docker

I solved the same issue in my container, by using the "-privileged" flag when doing "docker run", and then following these instructions from inside the container:

```
sudo mkdir -p /dev/net
sudo mknod /dev/net/tun c 10 200
sudo chmod 600 /dev/net/tun
```

Test whether the TUN/TAP device is available:

	sudo cat /dev/net/tun 

If you receive the message cat: 

	/dev/net/tun: File descriptor in bad state 

your TUN/TAP device is ready for use.

If you receive the message cat: 

	/dev/net/tun: No such device the TUN/TAP 

device was not successfully created: contact VPSLink Support for assistance.

## disable restart-always docker 

	docker update --restart=no my-container

Use the below to disable ALL auto-restarting (daemon) containers.

	docker update --restart=no $(docker ps -a -q)

---
layout: post
title: "Service Discovery under Docker Swarm Mode"
date:   2017-04-12
author: "@ajeetsraina"
tags: [linux,operations]
categories: beginner
img: "swarm-service-discovery.png"
terms: 2
---
Service Discovery under Swarm Mode.

## Purpose

The purpose of this lab is to illustrate how Service Discovery works under Swarm Mode.

## The application

WordPress is an open-source content management system (CMS) based on PHP and MySQL. 
It is a very simple containers application often used for demo purposes during meetup and conferences.


## Init your swarm

Let's create a Docker Swarm first

```.term1
docker swarm init --advertise-addr $(hostname -i)
```

From the output above, copy the join command (*watch out for newlines*) and paste it in the other terminal.

## Show members of swarm

From the first terminal, check the number of nodes in the swarm.

```.term1
docker node ls
$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
sxn3hrguu3n41bdt9srhqbnva *  node1     Ready   Active        Leader
tuhf7tmw4vxs3zq141x4ccgqg    node4     Ready   Active
w8dwf7sbgv3sa1otwy4ptqa5p    node3     Ready   Active
xsuyuqj2v254dy09t00w8we0t    node2     Ready   Active
```

For example, the above command shows 4 nodes, the first one being the manager, and the rest being workers.

## Create an overlay network

```.term1
docker network create -d overlay net1
```

### Verify that the new network gets created under swarm scope


```.term1
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
f35f0f62b82f        bridge              bridge              local
489b8399dbb6        docker_gwbridge     bridge              local
c3fc0f9aa474        host                host                local
u2q7yajhdfoo        ingress             overlay             swarm
dqczrsdaig1h        net1                overlay             swarm
88614c7df9c8        none                null                local
```

The output should show the newly added network called "net1" holding swarm scope .


### Creating MYSQL service

```.term1
docker service create \
           --replicas 1 \
           --name wordpressdb \
           --network net1 \
           --env MYSQL_ROOT_PASSWORD=mysql123 \
           --env MYSQL_DATABASE=wordpress \
          mysql:latest
```

The output should be like the following one (your ID should be different though).

```
ID            NAME                     MODE        REPLICAS  IMAGE
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE
obuppwh76qfn        wordpressdb         replicated          1/1                 mysql:latest
```

Let's list the tasks of the wordpressdb service.

```.term1
docker service ps wordpressdb
```

You should get an output like the following one where the 2 tasks (replicas) of the service are listed.

```.term1
$  docker service ps wordpressdb
ID                  NAME                IMAGE               NODE                DESIRED STATE
      CURRENT STATE           ERROR               PORTS
wnhlu88p4ipn        wordpressdb.1       mysql:latest        node2               Running
      Running 5 minutes ago
```


## Creating WordPress service


```.term1
docker service create \
           --env WORDPRESS_DB_HOST=wordpressdb \
           --env WORDPRESS_DB_PASSWORD=mysql123 \
           --network net1 \
           --replicas 4 \
           --name wordpressapp \
           --publish 80:80/tcp 
           wordpress:latest
```

Let's list out the services:



```.term1
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE
bawmqm2hymnq        wordpressapp        replicated          4/4                 wordpress:late
st
obuppwh76qfn        wordpressdb         replicated          1/1                 mysql:latest
```

You should get the output shown above where it shows both the service up and running with desired replicas of the containers.

Let's list the tasks of the wordpressapp service.

```.term1
$ docker service ps wordpressapp
ID                  NAME                IMAGE               NODE                DESIRED STATE
      CURRENT STATE           ERROR               PORTS
z8afhri573j6        wordpressapp.1      wordpress:latest    node4               Running
      Running 6 minutes ago
uuintj34xy2f        wordpressapp.2      wordpress:latest    node2               Running
      Running 6 minutes ago
xs1tijusm4u6        wordpressapp.3      wordpress:latest    node1               Running
      Running 6 minutes ago
okboroerec73        wordpressapp.4      wordpress:latest    node3               Running
      Running 6 minutes ago
```
### Service Discovery

Let us try to discover wordpressdb service from within one of wordpressapp container. Open up the manager node instance and run the below command:

```.term1
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS
           PORTS               NAMES
31b7b029334c        wordpress:latest    "docker-entrypoint..."   10 minutes ago      Up 10 min
utes       80/tcp              wordpressapp.3.xs1tijusm4u6eupxibkn2x2nk
```
There is a wordpressapp task running on the manager node(shown above). Let us enter into the wordpressapp.3 task(container) and 
try to reach out to wordpressab service using the service name.

```.term1
$ docker container exec -it 31b ping wordpressdb
PING wordpressdb (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: icmp_seq=0 ttl=64 time=0.060 ms
^C--- wordpressdb ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.060/0.060/0.060/0.000 ms
```
Voila ! We are able to ping one service to another using the service name.


{:.quiz}
What features are available under Swarm Mode?

- [] Service Discovery
- [] Load Balancing
- [] Routing Mesh
- [] Container Orchestration
- [] All of the above

---
title: "Visualizing a Docker Swarm Cluster"
date: 2017-12-08T18:49:43-05:00
draft: false
author: "Wes Shaddix"
tags: ["docker", "docker-swarm", "visualizer"]
---
# Goal

In a [previous blog post](http://www.wesshaddix.com/post/three-node-docker-swarm-cluster/) I setup a docker swarm cluster. Now I want a way to visualize what the swarm cluster looks like. How many nodes I have and which containers are running. Also as I change the state of the swarm cluster through docker commands I'd like to visualize what is actually happening.

## The solution

There is a docker image named `dockersamples/visualizer` that will give you a url that you can go to in order to view your swarm cluster. We will run this as a serivce on our cluster and bind it to run on **just the manager node** and not on the workers.

### Open up a port to host the visualizer website on

If you recall from the [previous blog post](http://www.wesshaddix.com/post/three-node-docker-swarm-cluster/) we setup our network only allow ssh traffic on port 22. We need to allow an additional port so that we are able to get to the visualizer service once we install it. In order to do that run the following command.

```powershell
az network nsg rule create -g docker-us-east --nsg-name nsg-docker-swarm -n allow-http-8080 --destination-port-ranges 8080 --access Allow --description "Allow inbound http 8080 traffic" --priority 101
```

### Install the visualizer service

Connect to your docker swarm manager node and setup a new service using the visualizer image.

```powershell
docker service create \
  --name=viz \
  --publish=8080:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer
```

### Verify that it works

If you navigate to <http://your-swarm-cluster:8080> you will see what it looks like. Note that you can use either the manager or either of the worker's public ip addresses to reach the service. Docker swarm ensures that all services are available on all of the ip addresses.

## Cleaning up

As before, the following command from the azure cli will deprovision the entire swarm cluster and all it's resources so that you don't get charged for it.

```powershell
az group delete -n docker-us-east
```

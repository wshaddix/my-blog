---
title: "Visualizing a Docker Swarm Cluster"
date: 2017-12-31T18:49:43-05:00
draft: false
author: "Wes Shaddix"
tags: ["docker", "docker-swarm", "visualizer"]
---

**[Updated] 12/31 - Updated instructions to leverage the load balancer configuration that we added to the swarm cluster.**

# Goal

In a [previous blog post](http://www.wesshaddix.com/post/three-node-docker-swarm-cluster/) I setup a docker swarm cluster. Now I want a way to visualize what the swarm cluster looks like. How many nodes I have and which containers are running. Also as I change the state of the swarm cluster through docker commands I'd like to visualize what is actually happening.

## The solution

There is a docker image named `dockersamples/visualizer` that will give you a url that you can go to in order to view your swarm cluster. We will run this as a service on our cluster and bind it to run on **just the manager node** and not on the workers.

### Install the visualizer service

Connect to your docker swarm manager node and setup a new service using the visualizer image. We are going to expose the visualizer on port 80 which we already opened up in both the network security group and the load balancer from the previous post.

```powershell
docker service create \
  --name=viz \
  --publish=80:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer
```

### Verify that it works

If you navigate to <http://your-swarm-cluster> you will see what it looks like.
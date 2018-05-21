---
title: "Upgrading Ubuntu on Docker Swarm Node"
date: 2018-05-21T09:53:37-04:00
lastmod: 2018-05-21T09:53:37-04:00
draft: false
keywords: []
description: "How to upgrade the OS of docker swarm nodes from Ubuntu 17.10 to Ubuntu 18.04"
tags: ["docker", "ubuntu", "docker-swarm"]
categories: []
author: "Wes Shaddix"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
---
In this post I'm going to document the steps needed to upgrade the OS of docker swarm nodes from Ubuntu 17.10 to Ubuntu 18.04
<!--more-->
# Scenario
From time to time you'll need to update the OS of the docker swarm nodes. In my case I'm upgrading from Ubuntu 17.10 to Ubuntu 18.04. You have to be cautious of two things when your host is in a docker swarm node. First you have to drain the node that is being updated so that no services are running on the node (and thus disappear) and second you have to make sure you never lose quorum on the swarm cluster by removing a manager node. **To maintain quorum, a majority of managers must be available.**

## Step 1: Evaluate Manager Status
The first step is to evaluate the manager status of the node. If the node is not a manager then you don't have to worry about losing the quorum when you take the node offline. If the node is a manager, then you need to promote another node in order to take over for the node that you're about to update.

`docker node promote <worker node name>`

`docker node demote <manager node name>`

## Step 2: Drain the Node
The next step is to drain the node so that the services currently running on that node are moved to other nodes in the swarm.

`docker node update --availability drain <node name>`

You can verify the status by running

`docker node ls`

and you should see that the `AVAILABILITY` of the node is `Drain`

You can verify that no containers are running on the node by running

`docker container ps`

## Step 3: Perform the OS upgrade
From the drained node perform the OS upgrade by running

`sudo do-release-upgrade`

Follow the prompts and answer the questions. Eventually you'll be prompted to restart the computer.

## Step 4: Add the new Docker Repository
Now that you have a new Ubuntu version you need to update the docker repository so that you can pull in the latest updates for your new version of Ubuntu.

``` bash
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

followed by

`sudo apt-get update`

## Step 5: Activate the Node
Set the node back to active in the swarm cluster by running the following command on a manager node

`docker node update --availability Active <node name>`

You can verify the status by running

`docker node ls`

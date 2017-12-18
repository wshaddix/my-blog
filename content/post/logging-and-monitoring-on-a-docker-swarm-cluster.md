---
title: "Logging and Monitoring on a Docker Swarm Cluster"
author: "Wes Shaddix"
tags: ["docker", "docker-swarm", "logging", "metrics", "sematext"]
date: 2017-12-12T19:11:04-05:00
draft: false
---

How to setup data analytics, metrics and events collection and logging on a docker swarm cluster using sematext SPM and Logsene.

<!--more-->

# Overview

After [setting up a docker swarm cluster](http://www.wesshaddix.com/post/three-node-docker-swarm-cluster/) and then being able to [visualize what is running on the cluster](http://www.wesshaddix.com/post/visualizing-a-docker-swarm-cluster/) the next concept I wanted to implement was logging and metrics. I've used several 3rd party logging and metrics services in the past but while reading [Docker Management Design Patterns](https://www.amazon.com/Docker-Management-Design-Patterns-Services/dp/148422972X) I found a great all-in-one solution called [Sematext](https://sematext.com/).

They have a service that includes logging, metrics, events and alerts and they have a docker agent that gets everything going pretty quickly. You simply sign up for an account, create your application and use your application tokens to configure your docker agent within your swarm cluster. The agent runs on each node of the swarm cluster.

## Create your Sematext account

The first step is to head over to [Sematext's registration page](https://apps.sematext.com/ui/registration) and create yourself an account.

## Create your application

Next head over to the [docker integration page](https://apps.sematext.com/ui/integrations/create/docker) and create a new application for the cluster. Be sure to also include logging support (checked by default)

![screenshot](/img/sematext-new-app.png)

## Configure your swarm stack

To make things easy to configure and deploy I created a [GitHub repo](https://github.com/wshaddix/docker-swarm-config) that stores my stack definition. So far I've included the vizualizer and the sematext docker agent as one stack as shown below.

```yml
version: '3.4'

services:
  vizualizer:
    image: dockersamples/visualizer
    volumes:
      - type: bind
        source: /var/run/docker.sock
        target: /var/run/docker.sock
    ports:
      - "8080:8080/tcp"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
  sematext:
    image: sematext/sematext-agent-docker
    volumes:
      - type: bind
        source: /var/run/docker.sock
        target: /var/run/docker.sock
      - type: bind
        source: /
        target: /rootfs
      - type: bind
        source: /etc/localtime
        target: /etc/localtime
    deploy:
      mode: global
    environment:
      - SPM_TOKEN=<YOUR SPM TOKEN>
      - LOGSENE_TOKEN=<16149055-8e94-4a8d-be80-a68962c60730>
```

I know that the SPM_TOKEN and LOGSENE_TOKEN shouldn't be in a git repo. I'll address that later, this is for learning/demo purposes only :)

## Setting up your swarm stack

In order to deploy your swarm stack you can:

### ssh into your swarm manager node

```powershell
ssh -i c:\Users\wessh\.ssh\docker_rsa wshaddix@<mgr public ip address>
```

### Checkout your swarm configuration repo

```powershell
git clone <your git repo address>
cd <your repo dir>
```

### Deploy your swarm stack

```powershell
docker stack deploy -c swarm-services.yaml
```

## View metrics and logs

This is going to run the Sematext docker agent on each of the host nodes in the cluster. Once they start reporting back to Sematext you can view the results in the Sematext web ui.

### Metrics

![metrics](/img/sematext-metrics.png)

### Logs

![logs](/img/sematext-logs.png)
---
title: "Three Node Docker Swarm with Ubuntu 17.04 on Azure"
author: "Wes Shaddix"
tags: ["docker", "docker-swarm"]
date: 2017-12-28T17:31:03-05:00
draft: false
---

**[Updated] 12/28 - Improved the setup so that we can use a load balancer in front of the swarm nodes as well as improve the availability of the swarm nodes**

# Overview

I'm learning docker swarm mode and need a realistic simulation of setting up a multi-node cluster. Azure is an ideal platform to test things out. My goal is to create a docker swarm cluster within a **single** region. Creating a multi-region swarm cluster requires creating virutal networks in different regions and connecting those vnets through a vpn gateway which is not something I'm ready to tackle just yet.

## My setup

I'm running a windows 10 workstation with docker for windows.

```powershell
λ ver
Microsoft Windows [Version 10.0.16299.125]

λ docker --version
Docker version 17.09.1-ce, build 19e2cf6
```

## Getting the azure cli v2

I know I want to use the Azure CLI v2 in order to be able to manage my azure resources from the command line but I don't want to have to install/configure it along with whatever programming languages it requires. Instead I just want to use it from a docker image. In order to do that, I need to download and run it in a container so from my **powershell** terminal I run

```powershell
docker run --rm -v ${HOME}:/root -it azuresdk/azure-cli-python:latest
```

This will run a container that has the azure cli ready to go. The `--rm` argument means that the container will be removed once I exit from it. The `-v ${HOME}:/root` argument maps my home directory on my Windows 10 host to the /root folder inside the docker container. This means anything written to the /root folder in the container will be saved on my host machine and be available the next time I run the container. Finally the `-it` argument will put me at the terminal of the container where I can interact with it.

## Logging into my azure account

Now that I have access to the Azure CLI, I need to authenticate to my Azure account. To do that I run

```powershell
az login
```

When you run this command it will give you a url along with a code. Simply open the url in your web browser and paste in the code and it will authorize your Azure CLI to access your Azure account.

## The script

I'm going to show the entire script here that you can run from the azure cli for convenience. In the rest of this post I'll describe what each command does.

```bash
RESOURCE_GROUP="dev-docker-swarm-us-east"
VNET_NAME="vnet-docker-swarm"
SUBNET_NAME="subnet-docker-swarm"
NSG_NAME="nsg-docker-swarm"
LOAD_BALANCER_NAME="load-balancer-swarm-cluster"
OS_IMAGE="Canonical:UbuntuServer:17.04:17.04.201711210"
VM_SIZE="Standard_B2S"
ADMIN_USERNAME="wshaddix"
AVAILABILITY_SET_NAME="availability-set-swarm-nodes"

# create a resource group
az group create -l eastus -n $RESOURCE_GROUP

# create a network security group
az network nsg create -g $RESOURCE_GROUP -n $NSG_NAME

# create a virtual network
az network vnet create -g $RESOURCE_GROUP -n $VNET_NAME

# create a subnet
az network vnet subnet create -g $RESOURCE_GROUP -n $SUBNET_NAME --vnet-name $VNET_NAME --address-prefix 10.0.0.0/24 --network-security-group $NSG_NAME

# create a public ip address for the load balancer (front-end)
az network public-ip create -g $RESOURCE_GROUP -n $LOAD_BALANCER_NAME-ip

# create a load balancer
az network lb create -g $RESOURCE_GROUP -n $LOAD_BALANCER_NAME --public-ip-address $LOAD_BALANCER_NAME-ip --frontend-ip-name $LOAD_BALANCER_NAME-front-end --backend-pool-name $LOAD_BALANCER_NAME-back-end

# create a load balancer probe on port 80
az network lb probe create -g $RESOURCE_GROUP -n load-balancer-health-probe-80 --lb-name $LOAD_BALANCER_NAME --protocol tcp --port 80

# create a load balancer traffic rule for port 80
az network lb rule create -g $RESOURCE_GROUP -n load-balancer-traffic-rule-80 --lb-name $LOAD_BALANCER_NAME --protocol tcp --frontend-port 80 --backend-port 80 --frontend-ip-name $LOAD_BALANCER_NAME-front-end  --backend-pool-name $LOAD_BALANCER_NAME-back-end --probe-name load-balancer-health-probe-80

# create three NAT rules for port 22 (so we can ssh to each of the three nodes via the load balancer's public ip address)
for i in `seq 1 3`; do
  az network lb inbound-nat-rule create -g $RESOURCE_GROUP -n nat-rule-for-node-$i-ssh --lb-name $LOAD_BALANCER_NAME --protocol tcp --frontend-port 422$i --backend-port 22 --frontend-ip-name $LOAD_BALANCER_NAME-front-end
done

# allow port 22 (ssh) traffic into the network
az network nsg rule create -g $RESOURCE_GROUP -n allow-ssh --nsg-name $NSG_NAME --destination-port-ranges 22 --access Allow --description "Allow inbound ssh traffic" --priority 100

# allow port 80 (http) traffic into the network
az network nsg rule create -g $RESOURCE_GROUP -n allow-http --nsg-name $NSG_NAME --destination-port-ranges 80 --access Allow --description "Allow inbound http traffic" --priority 200

# create three virtual network cards and associate with the network security group and load balancer. bind each NIC to one of the ssh nat rules we created
for i in `seq 1 3`; do
  az network nic create -g $RESOURCE_GROUP -n node-$i-private-nic --vnet-name $VNET_NAME --subnet $SUBNET_NAME --lb-name $LOAD_BALANCER_NAME --lb-address-pools $LOAD_BALANCER_NAME-back-end --lb-inbound-nat-rules nat-rule-for-node-$i-ssh
done

# create an availability set with 3 fault domains and 3 update domains
az vm availability-set create -g $RESOURCE_GROUP -n $AVAILABILITY_SET_NAME --platform-fault-domain-count 3 --platform-update-domain-count 3

# generate ssh keys
ssh-keygen -t rsa -f ~/.ssh/docker_rsa -N ""

# create three virtual machines
for i in `seq 1 3`; do
  az vm create -g $RESOURCE_GROUP -n node-$i --ssh-key-value "~/.ssh/docker_rsa.pub" --nics node-$i-private-nic --image $OS_IMAGE --size $VM_SIZE --authentication-type ssh --admin-username $ADMIN_USERNAME --availability-set $AVAILABILITY_SET_NAME --os-disk-name node-$i-os-disk
done

```

## Create a resource group

In order to be able to easily organize and later remove any and all resources that are related to the docker swarm cluster I put everything in a resource group. Resource groups are bound to geographic locations so I created one that is close to my location. The location could be [anywhere that Azure supports](https://azure.microsoft.com/en-us/regions/).

## Create a network security group

We want to protect our network traffic with a firewall so we create a new network security group next. We need the network security group to exist before we create our virtual network and subnet so that we can associate the subnet to the network security group. That way, any rules that we create in the firewall will apply to everything connected to the subnet. This way we don't have to manage rules for each individual NIC that gets attached to the subnet but instead we can manage the subnet as a whole.

## Create a virtual network

We start off by creating a network that each of the swarm nodes will connect to. We also create a subnet within the network to isolate traffic between the nodes.

## Create a subnet

We create a subnet within the virtual network and associate it to the network security group that we created previously. Any rules that we create later on for the firewall will apply to everything in this subnet.

## Create a public ip address for the load balancer (front-end)

Now that we have a network setup we'll start creating a load balancer that will balance traffic between each node in the swarm cluster. There will only be one public ip to the swarm cluster and that will be the ip address of the load balancer.

## Create a load balancer

Next we create a load balancer and associate it with the public ip address that we just created. The load balancer has a front-end ip and a back-end ip address pool that it balances traffic between.

## Create a load balancer probe on port 80

The load balancer has to have a way to detect if the nodes are healthy that it sends traffic to. We create a probe that will connect with each node using tcp port 80 traffic. If the node responds then the load balancer knows that the node is ready to receive traffic.

## Create a load balancer traffic rule for port 80

With the health probe in place, next we setup a traffic rule in the load balancer to forward tcp port 80 traffic to the nodes that are in the back-end pool. We associate this traffic rule with the health probe that we created.

## Create three NAT rules for port 22 (so we can ssh to each of the three nodes via the load balancer's public ip address)

To finish out the load balancer setup we need to create NAT rules that will allow us to reach each individual node in the cluster when we need to manage it via ssh. In order to do this, we setup rules that will route tcp port 4221, 4222 and 4223 traffic to port 22 (the ssh port) in the back-end pool. The key thing is that we name each NAT rule with a unique name that we will later use to bind to the NICs of the swarm nodes. This is what will allow us to ssh to port 4221, 4222 and 4223 and connect to node 1, node 2 and node 3 behind the load balancer.

## Allow port 22 (ssh) traffic into the network

In order to get tcp port 22 traffic into the network we have to allow for it in the firewall (network security group or NSG).

## Allow port 80 (http) traffic into the network

Just like port 22 ssh traffic above, we also have to let tcp port 80 traffic into the network.

## Create three virtual network cards

In order to connect the swarm nodes together we have to create a NIC for them and associate the NICs with the subnet that we previously setup. We give each NIC a unique name, we place it into the load balancer's back-end pool and we also associate the NIC with the uniquely named inbound NAT rule that we setup in order to allow us to reach the node via the load balancer for ssh traffic.

## Create an availability set with 3 fault domains and 3 update domains

The virtual machines that we setup as swarm nodes must all be in the same availability set in order for the load balancer to be able to route traffic to them. You can read more about availability sets [here](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/manage-availability?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json) but in essence, what an availability set does is gives you a way to ensure that not all of your vm nodes will be rebooted at the same time (update domains) and provisions them across different physical hardware (fault domains) in order to increase their availability and up-time. In a future post I'd like to explore using the new preview feature of [Availability Zones](https://docs.microsoft.com/en-us/azure/availability-zones/az-overview) to further improve the fault tolerance of the swarm cluster.

## Generate ssh keys

In order to manage the virtual machines we generate an ssh keypair so that each vm can be managed by the same ssh key for ease of administration.

## Create three virtual machines

Finally we can provision the virtual machines. For each VM we give it a unique name that aligns with the name of it's NIC and OS disk. We place it in the availability set that we created and associate it with the public key of the ssh key pair that we generated previously.

## Install docker on all three nodes

**On all three nodes** you have to install docker. Repeat these commands on each node via an ssh session.

### SSH to the specific node

In order to reach each individual node via the load balancer you have to ssh to the correct port based on how we setup the NAT rules earlier.

```powershell
# node 1
ssh -i ~/.ssh/docker_rsa wshaddix@<load balancer ip address> -p 4221

# node 2
ssh -i ~/.ssh/docker_rsa wshaddix@<load balancer ip address> -p 4222

# node 3
ssh -i ~/.ssh/docker_rsa wshaddix@<load balancer ip address> -p 4223
```

### Setup the docker apt repository

```powershell
sudo apt-get update

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

### Install Docker CE

```powershell
sudo apt-get update

sudo apt-get install docker-ce=17.09.1~ce-0~ubuntu
```

### Verify that docker is working correctly

```powershell
sudo docker run hello-world
```

### Manage docker as a non-root user

Right now anytime you interact with the docker cli you have to prefix the `docker` command with `sudo`. If you want
 to allow your user to run the docker cli as non-root you can create a docker group and add your user to it. Note that you'll have to log out and back in for this change to take effect.

```powershell
sudo usermod -aG docker $USER
```

### Configure docker to start on boot

```powershell
sudo systemctl enable docker
```

## Initialize the docker swarm

Make sure you run the following command on the **swarm manager node**

```powershell
docker swarm init --advertise-addr 10.0.0.4
```

The output of the above command will tell you what you need to run on the worker1 and worker2 nodes to join to the cluster.

## Add the worker nodes to the cluster

On the worker nodes run the command (my example is below) indicated when you initialized the swarm cluster from the previous step.

```powershell
docker swarm join --token <your swarm cluster's join token> 10.0.0.4:2377
```

## Verify the swarm cluster is up

On the manager node run the following command to ensure that the swarm is running with three nodes.

```powershell
docker node ls
```

## Cleaning up

After you are done and ready to deprovision all of the resources that we've created so that you are not billed for them, you can simply delete the resource group.

```powershell
az group delete -n dev-docker-swarm-us-east
```

You also may want to delete your docker_rsa ssh keypair if you are going to re-run these steps in the future.

```powershell
rm ~/.ssh/docker_rsa*
```

You may (or may not) also want to remove the three nodes from your `~/.ssh/known_hosts` file

## Saving time

If you are creating and removing swarm clusters frequently it is faster to run all of the commands at once as opposed to copy/paste/running each one. I've setup gists to speed up the process.

### Create the entire three node swarm cluster on Azure

After you've logged into your Azure account you can run

```powershell
wget -O - https://gist.githubusercontent.com/wshaddix/3acf084908f31d7c18f1f20f53b19147/raw/bf4ea5aa97a92020be50cd1c0cc2f7c7a6b65fbc/Setting%2520up%2520docker%2520swarm%2520on%2520azure | bash
```

This will setup the three virtual machines.

### Install Docker

Next, you can ssh into each of the three nodes and run

```powershell
wget -O - https://gist.githubusercontent.com/wshaddix/91058d471c525050f005e98eda85e3d3/raw/46de5d48bd4c4e51a9701e743c85fbf4dc3a3ce4/Install%2520Docker%2520on%2520Ubuntu%252017.04 | bash
```

### Initialize swarm mode

Now you've got your virtual machines provisioned and docker installed you can just run the commands listed above to initialize the swarm and join the worker nodes.
---
title: "Docker Swarm with Ubuntu Zesty 17.04 on Azure"
author: "Wes Shaddix"
tags: ["docker", "docker-swarm"]
date: 2017-12-06T17:31:03-05:00
draft: false
---

I'm learning docker swarm mode and need a realistic simulation of setting up a multi-node cluster. Azure is an ideal platform to test things out. My goal is to create a docker swarm cluster within a **single** region. Creating a multi-region swarm cluster requires creating virutal networks in different regions and connecting those vnets through a vpn gateway which is not something I'm ready to tackle just yet.

### My setup
I'm running a windows 10 workstation with docker for windows.
```powershell
λ ver
Microsoft Windows [Version 10.0.16299.98]

λ docker --version
Docker version 17.09.0-ce, build afdb6d4
```

### Getting the azure cli
I know I want to use the Azure CLI v2 in order to be able to manage my azure resources from the command line but I don't want to have to install/configure it along with whatever programming languages it requires. Instead I just want to use it from a docker image (I already know this exists). In order to do that, I need to download and run it in a container so from my powershell terminal I run
```powershell
docker run --rm -v ${HOME}:/root -it azuresdk/azure-cli-python:latest
```
This will run a container that has the azure cli ready to go. The `--rm` argument means that the container will be removed once I exit from it. The `-v ${HOME}:/root` argument maps my home directory on my Windows 10 host to the /root folder inside the docker container. This means anything written to the /root folder in the container will be saved on my host machine and be available the next time I run the container. Finally the `-it` argument will put me at the terminal of the container where I can interact with it.

### Logging into my azure account
Now that I have access to the Azure CLI, I need to authenticate to my Azure account. To do that I run
```powershell
az login
```
When you run this command it will give you a url along with a code. Simply open the url in your web browser and paste in the code and it will authorize your Azure CLI to access your Azure account.

### Creating a resource group
In order to be able to easily organize and later remove any and all resources that are related to my docker swarm I want to put everything in resource groups. Resource groups are bound to geographic locations so I will create one group that is close to my location. The location could be [anywhere that Azure supports](https://azure.microsoft.com/en-us/regions/).
```powershell
az group create -l eastus -n docker-us-east
```

### Creating the virtual network
Next up, we need a virtual network that the docker hosts can connect to in order to communicate with each other.
```powershell
az network vnet create -g docker-us-east -n docker-swarm-vnet
```

### Creating the network security group
In order have the ability to manage what network traffic want to allow or deny on our new virtual network we will create a security group.
```powershell
az network nsg create -g docker-us-east -n nsg-docker-swarm
```

### Allowing ssh traffic to our virtual network
Now we want to allow public ssh traffic onto our virtual network so that we are able to ssh and manage the docker nodes that we'll provision later.
```powershell
az network nsg rule create -g docker-us-east --nsg-name nsg-docker-swarm -n allow-ssh --destination-port-ranges 22 --access Allow --description "Allow inbound ssh traffic" --priority 100
```
### Creating the subnet
We need to create a subnet so that the docker nodes are able to communicate with each other. This subnet will also be protected by the `nsg-docker-swarm` network security group that we just created.
```powershell
az network vnet subnet create -g docker-us-east --vnet-name docker-swarm-vnet -n docker-subnet --address-prefix 10.0.0.0/24 --network-security-group nsg-docker-swarm
```

### Creating the public static ip addresses
In order for us to be able to ssh to the docker nodes over the internet we need a public ip address that we can assign to the host virtal machines.
```powershell
az network public-ip create -g docker-us-east -n docker-swarm-mgr-ip --allocation-method Static

az network public-ip create -g docker-us-east -n docker-swarm-worker1-ip --allocation-method Static

az network public-ip create -g docker-us-east -n docker-swarm-worker2-ip --allocation-method Static
```
The output of this command will tell you what ip addresses were allocated.

### Creating the NICs for the public connections of the docker nodes
The virtual machines will need to have a NIC for the public ip address that we created in the previous step.
```powershell
az network nic create -g docker-us-east -n docker-swarm-mgr-public-nic --vnet-name docker-swarm-vnet --subnet docker-subnet --public-ip-address docker-swarm-mgr-ip

az network nic create -g docker-us-east -n docker-swarm-worker1-public-nic --vnet-name docker-swarm-vnet --subnet docker-subnet --public-ip-address 
docker-swarm-worker1-ip

az network nic create -g docker-us-east -n docker-swarm-worker2-public-nic --vnet-name docker-swarm-vnet --subnet docker-subnet --public-ip-address docker-swarm-worker2-ip
```

### Generate ssh keys
We want to generate an ssh keypair so that we can access the docker nodes after we provision them
```powershell
ssh-keygen -t rsa -f ~/.ssh/docker_rsa -N ""
```

### Create the docker swarm manager
We'll create an Ubuntu 17.04 VM and attach the public and private NICs that we just created. This will join it to the subnet that we created.
```powershell
az vm create -g docker-us-east -n docker-swarm-mgr --ssh-key-value "~/.ssh/docker_rsa.pub" --nics docker-swarm-mgr-public-nic --image Canonical:UbuntuServer:17.04:17.04.201711210 --size Standard_B2S --authentication-type ssh --admin-username wshaddix --os-disk-name docker-swarm-mgr-os-disk
```

### Verify ssh access to the docker swarm manager
Once the swarm manager vm is provisioned, ensure that things are setup correctly by starting an ssh connection with the vm.
```powershell
ssh -i c:\Users\wessh\.ssh\docker_rsa wshaddix@<mgr public ip address>
```

### Create the first docker swarm worker
```powershell
az vm create -g docker-us-east -n docker-swarm-worker1 --ssh-key-value "~/.ssh/docker_rsa.pub" --nics docker-swarm-worker1-public-nic --image Canonical:UbuntuServer:17.04:17.04.201711210 --size Standard_B2S --authentication-type ssh --admin-username wshaddix --os-disk-name docker-swarm-worker1-os-disk
```

### Verify ssh access to the first docker swarm worker
Once the swarm manager vm is provisioned, ensure that things are setup correctly by starting an ssh connection with the vm.
```powershell
ssh -i c:\Users\wessh\.ssh\docker_rsa wshaddix@<worker1 public ip address>
```

### Create the second docker swarm worker
```powershell
az vm create -g docker-us-east -n docker-swarm-worker2 --ssh-key-value "~/.ssh/docker_rsa.pub" --nics docker-swarm-worker2-public-nic --image Canonical:UbuntuServer:17.04:17.04.201711210 --size Standard_B2S --authentication-type ssh --admin-username wshaddix --os-disk-name docker-swarm-worker2-os-disk
```

### Verify ssh access to the second docker swarm worker
Once the swarm manager vm is provisioned, ensure that things are setup correctly by starting an ssh connection with the vm.
```powershell
ssh -i c:\Users\wessh\.ssh\docker_rsa wshaddix@<worker2 public ip address>
```

### Install docker on all three nodes
**On all three nodes** you have to install docker. Repeat these commands on each node via an ssh session.

#### Setup the docker apt repository
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

#### Install Docker CE
```powershell
sudo apt-get update

sudo apt-get install docker-ce
```

#### Verify that docker is working correctly
```powershell
sudo docker run hello-world
```

#### Manage docker as a non-root user
Right now anytime you interact with the docker cli you have to prefix the `docker` command with `sudo`. If you want
 to allow your user to run the docker cli as non-root you can create a docker group and add your user to it. Note that you'll have to log out and back in for this change to take effect.
```powershell
sudo usermod -aG docker $USER
```

#### Configure docker to start on boot
```powershell
sudo systemctl enable docker
```

### Initialize the docker swarm
Make sure you run the following command on the **swarm manager node**
```powershell
docker swarm init --advertise-addr 10.0.0.4
```
The output of the above command will tell you what you need to run on the worker1 and worker2 nodes to join to the cluster.

### Add the worker nodes to the cluster
On the worker nodes run the command (my example is below) indicated when you initialized the swarm cluster from the previous step.
```powershell
docker swarm join --token SWMTKN-1-490uiqnix2tsuknaj1g4nz95c9rw3ckuwmaafdpy4h59cnyu7a-21hx4q1v6i4i76lrndhssdcr5 10.0.0.4:2377
```

### Verify the swarm cluster is up
On the manager node run the following command to ensure that the swarm is running with three nodes.
```powershell
docker node ls
```

### Cleaning Up
After you are done and ready to deprovision all of the resources that we've created so that you are not billed for them, you can simply delete the resource group.
```powershell
az group delete -n docker-us-east
```
You also may want to delete your docker_rsa ssh keypair if you are going to re-run these steps in the future.
```powershell
rm ~/.ssh/docker_rsa*
```
You may (or may not) also want to remove the three nodes from your `~/.ssh/known_hosts` file
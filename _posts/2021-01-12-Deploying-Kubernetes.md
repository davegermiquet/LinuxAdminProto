---
layout: post
title: Deploying Kubernetes/Calico on AWS
subtitle: Implementation Jenkins/Terraform/Ansible
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [Kubernetes, Calico, AWS, Jenkins, Terraform ]
---
[10]: <#decisionreason>
[20]: <#prepareaws>
[30]: <#finalconfig>
[40]: <#jenkins>
[50]: <#terraform>
[60]: <#squid>
[70]: <#kubemaster>
[80]: <#sslsetup>
[90]: <#kubenode>
[100]: <#cniplugin>
[110]: <#typha>
[120]: <#calico>
[130]: <#troubleshooting>
[140]: <https://github.com/davegermiquet/kubernetesautodeploy>

We will be going over the following steps:
- [Final Configuration After Implementation][30]
- [Technologies Used][10]
- [Preparing AWS For Terraform Deployment][20]
- [Setting up Jenkins][40]
- [Terraform Description/ How it Works][50]
- [Deploying Squid][60]
- [Deploying the Master Kubernetes][70]
- [Settings up the SSL Certificates][80]
- [Deploying the Node Kubernetes][90]
- [Install the CNI Plugin][100]
- [Install the Typha Container][110]
- [Installing Calico Container][120]
- [TroubleShooting and Debugging][130]


<a name="finalconfig"></a>
### **Final Configuration After Implementation**

|Components Installed | Description |
| ----------- | ----------- |
| Docker   |  Container/Linux System |
| Kubernetes  |  Cluster/Management System (manages docker)|
| Felix | Main task: Programs routes and ACLs, and anything else required on the host to provide desired connectivity for the endpoints on that host. Runs on each machine that hosts endpoints. Runs as an agent daemon|
| (Calico CNI PLugin) | The Calico binary that presents this API to Kubernetes is called the CNI plugin, and must be installed on every node in the Kubernetes cluster. The Calico CNI plugin allows you to use Calico networking for any orchestrator that makes use of the CNI networking specification. Configured through the standard CNI configuration mechanism, and Calico CNI plugin.|
| IPAM plugin| Main task: Uses Calico’s IP pool resource to control how IP addresses are allocated to pods within the cluster. It is the default plugin used by most Calico installations. It is one of the Calico CNI plugins.|
| kube-controllers |Main task: Monitors the Kubernetes API and performs actions based on cluster state. kube-controllers.|
| Typha | Main task: Increases scale by reducing each node’s impact on the datastore. Runs as a daemon between the datastore and instances of Felix. Installed by default, but not configured. Typha description, and Typha component.|
|calicoctl|Main task: Command line interface to create, read, update, and delete Calico objects. calicoctl command line is available on any host with network access to the Calico datastore as either a binary or a container. Requires separate installation. calicoctl.|

Severs Deployed:

- 2 EC2 Instances (Running on Ubuntu)
- Squid (so private network can contact and download packages or images)
- Kubernetes Master
- Kubernetes Node
- VPC Network With only specific ports open for kubernetes
- A Private and Public Network
- Node On Private Network no access to outside world except through squid
- Master on public network

**You will want to fork the following repository and configure it instead of using mine, as my project uses my own S3 bucket which will have to be modified in the terraform file: 
[https://github.com/davegermiquet/kubernetesautodeploy][140]**


<a name="decisionreason"></a>
### **Technologies Used To Deploy**

- AWS (EC2)
- Jenkins
- Terraform
- Ansible


<a name="prepareaws"></a>
### **Preparing AWS For Terraform Deployment**

In order to prepare your AWS System you would need to do the following:

- Create an API key/password with AWS, and enable it inside jenkins

<a name="jenkins"></a>
### **Setting up Jenkins**

#### Pre-Requisites:
- docker, docker-compose

The application "docker-compose" is a utility that spins up a docker instance configured to the application and pre-bundled making things
easier to spin up and use.

For testing purposes you can set up jenkins using your desktop, with the following git hub repository:
```
# git clone https://github.com/davegermiquet/jenkinsautodeploy.git
```
One you have checkout the repository run the following:

```
docker-compose -f docker-compose-jenkins.yml up -d

```

Log on to the docker instance and create a generic SSH key which all instances can talk to each other for installation with ansible.
Terraform takes this key and distrubutes it to all EC2 Instances that will be deployed, by an environment variable will discuss under terraform

```
docker-compose -f docker-compose-jenkins.yml exec jenkins /bin/bash
```

```
jenkins@c5767258f978:/$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/var/jenkins_home/.ssh/id_rsa): 
Created directory '/var/jenkins_home/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /var/jenkins_home/.ssh/id_rsa.
Your public key has been saved in /var/jenkins_home/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:ACwn3YaH28YA81Y6CDL/5yIYjj+uAPunjJeoT1MFLPg jenkins@c5767258f978
The key's randomart image is:
+---[RSA 2048]----+
|+.o=o+.          |
|o+++O=+          |
| .o==Bo          |
|  Eo.o+.         |
|o   o.. S        |
|++ . o           |
|=o+.. .          |
|o*+o..           |
|==B+             |
+----[SHA256]-----+
```


Once deployed, browse to the box you ran docker on and go here:

http://ipaddress:8888/


Do the following steps:

Select Manage Jenkins
Select Manage Credentials
Add the following as your credentials (AWS KEY/PASS) pair.
Use the ID as "AMAZON_CRED"

![image-title-here](/assets/img/jenkinspage4.jpg){:class="img-responsive"}

After Adding your credentials:

Select New Item Option
Select Multi Branch Pipeline
![image-title-here](/assets/img/jenkinsfirstpage.jpg){:class="img-responsive"}
Enter Your Fork Repository under the Branch
![image-title-here](/assets/img/jenkinssecondpage.jpg){:class="img-responsive"}

<a name="terraform"></a>
### **Breaking down Terraform configuration file**

Terraform is an application that can create infrastructure for networks, computers, user levels, firewalls ... , in code, and it is persistent and stays the same,
as long as no changes is done. I've configured the state to be aware on the S3 bucket, so it knows what to destroy, when I want to destroy the entire infrastructure.


In between the terraform {} are the directives telling the terraform application what to do:
```
backend "s3" {
    bucket = "terraforms3state"
    key    = "autodeploy"
    region = "us-east-1"
}

required_providers {
    aws = {
    source  = "hashicorp/aws"
    version = "~> 2.70"
    }
}

provider "aws" {
profile = "default"
region  = "us-east-1"
}
```

This is where you define where you want to deploy the infrastructure and the s3 bucket. You will need to change these settings for your own particular AWS
account.


<a name="squid"></a>
### **Deploying Squid**

<a name="kubemaster"></a>
### **Deploying the Master Kubernetes**

<a name="sslsetup"></a>
### **Settings up the SSL Certificates**


<a name="kubenode"></a>
### **Deploying the Node Kubernetes**<a name="Install CNI Plugin"></a>
<a name="cniplugin"></a>
### **Install the CNI Plugin**

<a name="typha"></a>
### **Install the Typha Container**<a name="typha"></a>
### **Installing Calico Container**
<a name="calico"></a>

<a name="troubleshooting"></a>
## **TroubleShooting and Debugging**
### **TroubleShooting and Debugging**
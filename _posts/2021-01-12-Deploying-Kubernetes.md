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
[150]: <#jenkinspre>
[160]: <#jenkinsexplained>
[170]: <#resetstate>
We will be going over the following steps:
- [Final Configuration After Implementation][30]
- [Technologies used and why I used this design][10]
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

### Summary on components:

#### Squid

The reason why I choose to install squid on the AWS gateway network, is so the private network can contact and install the latest updates, without having direct access to the Global Internet
I save money on not using the AWS NAT gateway prices, by creating my own squid, and the nodes are segregated on a private network
making more security.
It also allows it CLOUD independence.

#### Terraform Templates and Jinja

The reason why I choose to use Ninja templates was it was an easy way to customize terraform where environment variables are not allowed, and I was able to dynamically choose those nodes with that logic

#### SSH Tunneling

I choose SSH tunneling, so I could run ansible on all nodes, and I copy the jenkins certificate for EASE Of use to all the nodes.






<a name="prepareaws"></a>
### **Preparing AWS For Terraform Deployment**

In order to prepare your AWS System you would need to do the following:

- Create an API key/password with AWS, and enable it inside jenkins

<a name="jenkins"></a>

We will be going over the following steps:
- [Pre-Setup Jenkisn][150]
- [Resetting the AWS state][170]
- [Jenkinsfile Explained and how it works][160]
### **Setting up Jenkins**

<a name="jenkinspre"></a>
#### **Pre-Requisites:**
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

![image-title-here](/assets/img/jenkinspage4.jpg)

After Adding your credentials:

Select New Item Option
Select Multi Branch Pipeline
![image-title-here](/assets/img/jenkinsfirstpage.jpg)
Enter Your Fork Repository under the Branch
![image-title-here](/assets/img/jenkinssecondpage.jpg)

###### Specific Settings you need to add:

2 required fields specific to your install is:

BUCKET: The initial S3 bucket on your amazon account where terraform will store its state files
domain: (your domain on AWS for Multi Region Setup, so you can connect them)

###### Suggestions:

The following field: RED_HAT_IMAGE_AMI

This build requires REDHAT type (yum package installations), it should work with Redhat 8 and Centos 8 find the AMIs.
Here is link that could help:

https://wiki.centos.org/Cloud/AWS

If your changing the AMI image, you'll probably need to change this field as well:

USER_AWS

For centos that user would be "centos" not ec2-user

<a name="resetstate"></a>

1. Run the build with the following settinsg:


TASK = DESTROY
DESTROY_NODES = Yes (Erase NODES first)
Setup the DOMAIN/USER/AMI appropriately

2. Run the build with the following settinsg:


TASK = DESTROY
DESTROY_NODES = NO (Erase NODES first)
Setup the DOMAIN/USER/AMI appropriately

It should now be destroyed completely on AWS


<a name="jenkinsexplained"></a>

#### **Jenkinsfile Explained Strategy**
```
agent any
environment {
TF_IN_AUTOMATION      = '1'
// Common SSH file for all amazon instances
// SSH key for jenkins system for public key should be located below for copying to Amazon Platforms
TF_VAR_SSH_PUB = readFile "/var/jenkins_home/.ssh/id_rsa.pub"
}
```

This section, is the pre-setup of the entire build to deploy Kubernetes, it sets up the following:
agent - which agent

Necessary environment variables:

| ENV VAR |Description |
| ------- | ----------- |
| TF_IN_AUTOMATION      = '1' | Terraform automated flag for deploiying infrastructure |
| TF_VAR_SSH_PUB = readFile "/var/jenkins_home/.ssh/id_rsa.pub" | read the SSH public key and copy over into terraform, and other ec2 instance for logging in|

```
parameters {
   choice(name: 'TASK', choices: ['apply','destroy'], description: 'deploy/destroy')
  choice(name: 'REDEPLOY_MASTER', choices: ['yes','no'], description: 'Redeploy Master and nodes if no it will only reconfigure nodes')
  choice(name: 'DESTROY_NODES', choices: ['no','yes'], description: 'Only Destroy Nodes')
  string(name: 'BUCKET', defaultValue : '',description: 'This is the S3 bucket Stateforms are saved please create on amazon')
  string(name: 'AWS_FIRST_REGION', defaultValue: 'us-east-1', description: 'First Region to Deploy for ec2 and s3 bucket')
  string(name: 'AWS_SECOND_REGION', defaultValue: 'us-west-1', description: 'Second Region to Deploy')
  string(name: 'DOMAIN', defaultValue:'',description: 'Domain for the kubernetes cluster(will create Route 53)')
  string(name: 'VPC_RANGE',defaultValue: '192.168.0.0/16',description: 'IP RANGE For VPC Created')
  string(name: 'IP_PUBILC_RANGE',defaultValue: '192.168.10.0/24',description: 'IP RANGE For Public Subnet')
  string(name: 'IP_PRIVATE_RANGE',defaultValue: '192.168.9.0/24',description: 'IP RANGE For Prviate Subnet')
  string(name: 'TYPE_EC2_INSTANCE',defaultValue: 't2.medium',description: 'EC2 Server Type')
  string(name: 'HARD_DISK_SIZE',defaultValue: '16',description: 'In Gigabytes')
  string(name: 'RED_HAT_IMAGE_AMI',defaultValue: 'ami-096fda3c22c1c990a',description: 'RedHat Deployment Image version 7')
  string(name: 'USER_AWS',defaultValue: 'ec2-user',description: 'user name to login as')
  string(name: 'NODE_AMOUNT',defaultValue: '2',description: 'Amount of Nodes Deployed')
          }

```

This section allows you to configure the terraform state S3 bucket, and variables to deploy the cluster.
The 2 fields YOU must specifically choose, to your unique deloyment is S3 Bucket and DOMAIN(example trulycanadian.net)

'''
stage('Create Kubernetes Terraform Files or master/node' ) {
when {  expression { REDEPLOY_MASTER=='yes' } }
steps {
sh 'ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -vv  -i inventory_hosts --user ec2-user --extra-vars "target=127.0.0.1" ${WORKSPACE}/playbooks/create_terraform_master.yml'
}
'''


##### Stage 1:

This stage uses ansible, to create the the kubenetes master template based on the inputs you choose in the parameters section. We will take about this in the ansible configuration files.

The field REDPLOY_MASTER  means it will only run if REDPLOY_MASTER is enabled.

```
stage('Create Environment for Kubernetes master and Docker')
steps {
withCredentials([usernamePassword(credentialsId: 'AMAZON_CRED', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
echo 'Deploying to DEV/QA AWS INSTANCE'
sh "cd terraform_master;terraform init  -input=false"
sh "cd terraform_master;terraform ${TASK} -input=false -auto-approve"
script {
server_deployed = sh ( script: 'cd terraform_master;terraform output KUBE_MASTER_PUBLIC_IP | sed "s/\\\"//g"', returnStdout: true).trim()
private_ip_deployed = sh ( script: 'cd terraform_master;terraform output KUBE_MASTER_PRIVATE_IP | sed "s/\\\"//g"', returnStdout: true).trim()
KEY_NAME_DEPLOYER = sh ( script: 'cd terraform_master;terraform output KEY_NAME_DEPLOYER | sed "s/\\\"//g"', returnStdout: true).trim()
SECURITY_GROUP_GLOBAL = sh ( script: 'cd terraform_master;terraform output SECURITY_GROUP_GLOBAL', returnStdout: true).trim()
PRIVATE_SUBNET = sh ( script: 'cd terraform_master;terraform output PRIVATE_SUBNET  | sed "s/\\\"//g"', returnStdout: true).trim()
IAM_INSTANCE_PROFILE = sh ( script: 'cd terraform_master;terraform output IAM_INSTANCE_PROFILE  | sed "s/\\\"//g"', returnStdout: true).trim()
}
}
}
}
```



##### Stage 2:

This stage sets up the infrastructure of the amazon instance using Terraform it creates the following:

- withCredentials function sets up the AWS API Key/Password for Terraform using environment variables
- init, initializes terraform --input=false makes it s its not interactive
- The Task is replaced with APPLY (for create and destroy for remove) which runs the procedure of the terraform yml to create/destroy saving a state file on a S3 bucket defined in the terraform file --auto-approve makes it so no confirmation happens on run

Script Section makes it so it can receive all the output variables for the following EC2 for debugging and use of ansible scripts:
- The Public IP to connect from your INTERNET jenkins
- The private IP for master for internally on EC2
- It also has OUTPUT variables that it uses to Create the nodes, it stores it in variables to be used later by the node process

I store these values in jenkins to pass to an ansible build to create a terraform node terraform file, that creates dynamic nodes.

##### Stage 3:
```
 stage('install packages on aws instance and squid instance') {
           environment {
              SERVER_DEPLOYED="${server_deployed}"
              PRIVATE_IP_DEPLOYED="${private_ip_deployed}"
              }
         when { expression { params.TASK == 'apply' && params.REDEPLOY_MASTER == 'yes' } }
         steps  {
              sh  '''
              echo "awsserver ansible_port=22 ansible_host=${SERVER_DEPLOYED}" > inventory_hosts
              ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -vv  -i inventory_hosts --user ec2-user --extra-vars "workspace=${WORKSPACE} target=awsserver" ${WORKSPACE}/playbooks/deploy-squid-playbook.yml
              '''
              }
           }
```

Description:

Run Ansible File For Squid (See Ansible Documentation for each file) Use the SERVER_DEPLOYED variable to connect and run the ansible playbook

##### Stage 4: Install and Deploy the Master
```
stage('install kubernetes master') {
            environment {
              SERVER_DEPLOYED="${server_deployed}"
              PRIVATE_IP_DEPLOYED="${private_ip_deployed}"
            }
              when {  expression { params.TASK == 'apply' && params.REDEPLOY_MASTER=='yes' } }
              steps  {
                sh  '''
                echo "awsserver ansible_port=22 ansible_host=${SERVER_DEPLOYED}" > inventory_hosts
                echo "kuber_node_1 ansible_port=2222 ansible_host=localhost" >> inventory_hosts
                mkdir -p etc/kubernetes/
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -vv  -i inventory_hosts --user ec2-user --extra-vars "kuburnetes_master=${PRIVATE_IP_DEPLOYED} workspace=${WORKSPACE} target=awsserver" ${WORKSPACE}/playbooks/install-kubernetes-master-playbook.yml
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -vv  -i inventory_hosts --user ec2-user --extra-vars "kuburnetes_master=${PRIVATE_IP_DEPLOYED} workspace=${WORKSPACE} target=awsserver" ${WORKSPACE}/playbooks/create-ssl-certs.yml
              '''
               }
              }
```

This Step Installs and Deploys the master on the first EC2 Instance. It sets up the environment  variables for the private ip and the public ip and the node. It sets up the inventory hosts file, creates a folder where we'll put the files generated by the master, and run the following playbooks which deploys the main instance of the kubernetes master and creates the ssl certificates.


```
       stage('install addons to master') {
              environment {
                SERVER_DEPLOYED="${server_deployed}"
                 PRIVATE_IP_DEPLOYED="${private_ip_deployed}"
                 CMD_TO_RUN="${cmd_to_join}"
                 TF_VAR_SSH_PUB = readFile "/var/jenkins_home/.ssh/id_rsa.pub"
                 HTTP_PROXY="http://${private_ip_deployed}:3128"
              }
              when {  expression { params.TASK == 'apply' &&  params.REDEPLOY_MASTER=='yes' } }
              steps {
              sh '''
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -vv  -i inventory_hosts --user ec2-user --extra-vars "http_ansible_proxy=${HTTP_PROXY} cmd_to_run=${CMD_TO_RUN} kuburnetes_master=${PRIVATE_IP_DEPLOYED} workspace=${WORKSPACE} target=awsserver" ${WORKSPACE}/playbooks/install-addons-kubernetes.yml
              '''
                }
            }

##### Stage 5: Install and Deploy the Master

This stage will deploy the CNI PLUGIN, settings, for Calico, to get it ready to setup the clustesd networking

```
stage('install typha to kubernetes master') {
              environment {
              SERVER_DEPLOYED="${server_deployed}"
              PRIVATE_IP_DEPLOYED="${private_ip_deployed}"
              CMD_TO_RUN="${cmd_to_join}"
              TF_VAR_SSH_PUB = readFile "/var/jenkins_home/.ssh/id_rsa.pub"
              HTTP_PROXY="http://${private_ip_deployed}:3128"
              }
              when {  expression { params.TASK == 'apply' && params.REDEPLOY_MASTER=='yes' } }
              steps {
                sh '''
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -vv  -i inventory_hosts --user ec2-user --extra-vars "http_ansible_proxy=${HTTP_PROXY} cmd_to_run=${CMD_TO_RUN} kuburnetes_master=${PRIVATE_IP_DEPLOYED} workspace=${WORKSPACE} target=awsserver" ${WORKSPACE}/playbooks/install-typha-calinco-node.yml
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -vv  -i inventory_hosts --user ec2-user --extra-vars "http_ansible_proxy=${HTTP_PROXY} cmd_to_run=${CMD_TO_RUN} kuburnetes_master=${PRIVATE_IP_DEPLOYED} workspace=${WORKSPACE} target=awsserver" ${WORKSPACE}/playbooks/install-calico-node.yml
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -vv  -i inventory_hosts --user ec2-user --extra-vars "http_ansible_proxy=${HTTP_PROXY} cmd_to_run=${CMD_TO_RUN} kuburnetes_master=${PRIVATE_IP_DEPLOYED} workspace=${WORKSPACE} target=awsserver" ${WORKSPACE}/playbooks/configure-felix-configure-bgp.yml
              '''
                 }
              }
```


##### Stage 6:  Install TYPHA Components and Node_calico and configure felix for Calico


This plugin activates and deploys the CALICO CNI plugin for the MASTER Kubernetes Nodes


##### Stage 7:  Install TYPHA Components and Node_calico and configure felix for Calico



# Tune in to next publication!


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
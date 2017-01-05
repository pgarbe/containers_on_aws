Running Containers on AWS
=========================
This repository contains some practical examples how to run containers on AWS. It covers Docker Swarm (mode), Docker for AWS and ECS.

## Basic setup

In order to run the different kind of clusters we need a basic setup (like VPC). There are some very good templates [available](https://github.com/widdix/aws-cf-templates/tree/master/vpc) from [Cloudonout](https://cloudonout.io) which I want to reuse.


```bash
# Create VPC in 3 availability zones
aws cloudformation create-stack \
  --stack-name vpc \
  --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-3azs.yaml

# Create bastion host for ssh access
aws cloudformation create-stack \
  --stack-name vpc-ssh-bastion \
  --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-ssh-bastion.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc ParameterKey=KeyName,ParameterValue=pgarbe
```


## Docker Swarm (mode)
To create a Docker swarm (mode) you need to setup managers and workers. A swarm cluster can be initialized by `docker swarm init` and further nodes can be joined by `docker swarm join`. In order to join nodes we've to provide a so-called join-token. This can be requested on the first node by `docker swarm join-token worker|manager`. 


```bash
aws cloudformation create-stack  \
  --template-body file://./swarm-mode/manager.yaml \
  --stack-name swarm-manager \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion ParameterKey=KeyName,ParameterValue=pgarbe ParameterKey=DockerVersion,ParameterValue=1.13.0~rc4 ParameterKey=DockerPreRelease,ParameterValue=true ParameterKey=DesiredInstances,ParameterValue=1

# ssh into node via bastion host
ssh -A ec2-user@<IP of bastion host>

# ssh into node via bastion host
ssh ubuntu@<IP of manager node>

# Initialize swarm cluster
docker swarm init

# Get the swarm join tokens and copy them
docker swarm join-token manager --quiet
docker swarm join-token worker --quiet

# Encrypt tokens with KMS
tbd

# Update stack to create more manager nodes
aws cloudformation update-stack  \
  --template-body file://./swarm-mode/manager.yaml \
  --stack-name swarm-manager \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion ParameterKey=KeyName,ParameterValue=pgarbe ParameterKey=DockerVersion,ParameterValue=1.13.0~rc4 ParameterKey=DockerPreRelease,ParameterValue=true ParameterKey=DesiredInstances,ParameterValue=3 ParameterKey=SwarmManagerJoinToken,ParameterValue={KmsEncryptedManagerToken}
```


## Docker for AWS
tbd

## ECS
tbd

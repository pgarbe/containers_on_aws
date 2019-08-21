Running Containers on AWS
=========================
This repository contains some practical examples how to run containers on AWS. It covers Docker Swarm (mode), Docker for AWS and ECS.

## Basic setup

In order to run the different kind of clusters we need a basic setup (like VPC). There are some very good templates [available](https://github.com/widdix/aws-cf-templates/tree/master/vpc) from [Cloudonaut](https://cloudonaut.io) which I want to reuse.


```bash
# Create VPC in 3 availability zones
aws cloudformation create-stack \
  --stack-name vpc \
  --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-3azs.yaml

# Create bastion host for ssh access
aws cloudformation update-stack \
  --stack-name vpc-ssh-bastion \
  --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-ssh-bastion.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc ParameterKey=KeyName,ParameterValue=pgarbe

# Create NAT gateway
aws cloudformation create-stack \
  --stack-name vpc-nat-instance \
  --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-nat-instance.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc \
               ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion \
               ParameterKey=KeyName,ParameterValue=pgarbe
```

## Single Docker (Ubuntu)

```bash
aws cloudformation create-stack  \
  --template-body file://./ubuntu/stack.yaml \
  --stack-name docker \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc \
               ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion \
               ParameterKey=KeyName,ParameterValue=pgarbe \
               ParameterKey=DockerVersion,ParameterValue=1.13.0~rc6 \
               ParameterKey=DockerPreRelease,ParameterValue=true \
               ParameterKey=DesiredInstances,ParameterValue=1
```

#### Deploy a service
```bash
docker run -d -p 80:80 --name nginx nginx
```

## Docker Swarm (mode)
To create a Docker swarm (mode) you need to setup managers and workers. A swarm cluster can be initialized by `docker swarm init` and further nodes can be joined by `docker swarm join`. In order to join nodes we've to provide a so-called join-token. This can be requested on the first node by `docker swarm join-token worker|manager`. 


```bash
./deploy.sh ParameterKey=KeyName,ParameterValue=pgarbe \
            ParameterKey=Version,ParameterValue=$(date +%s) 

# ssh into node via bastion host
ssh -A ec2-user@<Public IP of bastion host>

# ssh into node 
ssh ubuntu@<Private IP of manager node>

# Get the swarm join tokens and copy them
docker swarm join-token manager --quiet
docker swarm join-token worker --quiet

# Encrypt tokens with KMS
swarm_manager_join_token=$(aws kms encrypt --key-id <KmsKey> --plaintext <SwarmManagerJoinToken> --output text --query CiphertextBlob)
swarm_worker_join_token=$(aws kms encrypt --key-id <KmsKey> --plaintext <SwarmWorkerJoinToken> --output text --query CiphertextBlob)

./deploy.sh ParameterKey=KeyName,ParameterValue=pgarbe \
            ParameterKey=Version,ParameterValue=$(date +%s)  \
            ParameterKey=SwarmManagerJoinToken,ParameterValue=$swarm_manager_join_token \
            ParameterKey=SwarmWorkerJoinToken,ParameterValue=$swarm_worker_join_token
```

#### Deploy a service
```bash
docker stack deploy --compose-file docker-stack.yaml voting-app
```


## Docker for AWS
[Docker for AWS](https://www.docker.com/products/docker#/AWS) provides an easy-to-deploy Docker environment on AWS. The installation is very easy and takes only a couple of minutes.

```bash
aws cloudformation create-stack  \
  --template-url https://editions-us-east-1.s3.amazonaws.com/aws/stable/Docker.tmpl \
  --stack-name docker4aws113 \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ClusterSize,ParameterValue=5 \
               ParameterKey=EnableCloudWatchLogs,ParameterValue=yes \
               ParameterKey=EnableSystemPrune,ParameterValue=no \
               ParameterKey=InstanceType,ParameterValue=t2.micro \
               ParameterKey=KeyName,ParameterValue=pgarbe \
               ParameterKey=ManagerDiskSize,ParameterValue=20 \
               ParameterKey=ManagerDiskType,ParameterValue=standard \
               ParameterKey=ManagerInstanceType,ParameterValue=t2.micro \
               ParameterKey=ManagerSize,ParameterValue=3 \
               ParameterKey=WorkerDiskSize,ParameterValue=20 \
               ParameterKey=WorkerDiskType,ParameterValue=standard
```

Get the public IP from one of the manager nodes.

```bash
ssh docker@<Public IP of manager node>
```

#### Deploy a service

```bash
docker service create \
  --name=viz \
  --publish=8080:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  manomarks/visualizer

docker service create --publish 80:80 --name nginx nginx


```

## ECS
tbd

```bash
aws cloudformation create-stack  \
  --template-body file://./ecs/cluster.yaml \
  --stack-name ecs-cluster \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc \
               ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion \
               ParameterKey=KeyName,ParameterValue=pgarbe \
               ParameterKey=DesiredInstances,ParameterValue=3


aws cloudformation create-stack  \
  --template-body file://./ecs/service.yaml \
  --stack-name ecs-service \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc \
               ParameterKey=ParentECSStack,ParameterValue=ecs-cluster \
               ParameterKey=DesiredInstances,ParameterValue=2

```

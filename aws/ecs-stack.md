# Amazon Elastic Container Service Fargate

TODO:

* Config HTTPS for ELB.
* Pull image from private reposistory.
* Configure domain for application load balancer.
* Setup auto scaling.

This document provide the manual steps to deploy a flask RESTful application to Amazon ECS Fargate.

![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/amazon-ecs-fargate.jpg)

## Concepts and definition

### Amazon ELB

Elastic Load Balancing automatically distributes incoming application traffic across multiple targets, such as Amazon EC2 instances, containers, and IP addresses. It can handle the varying load of your application traffic in a single Availability Zone or across multiple Availability Zones. In this demostration we will use Application Load Balancer to distributes incoming application traffic to multiple Todo API containers within a single Availability Zone.

This [link](https://aws.amazon.com/elasticloadbalancing/) provide more details

### ECS Service

Amazon ECS allows you to run and maintain a specified number of instances of a task definition simultaneously in an Amazon ECS cluster. This is called a service. If any of your tasks should fail or stop for any reason, the Amazon ECS service scheduler launches another instance of your task definition to replace it and maintain the desired count of tasks in the service depending on the scheduling strategy used.

This [link](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html) provide more details

### VPC

Amazon Virtual Private Cloud is a virtual networking environment consist ELB, our application, services, etc... We can config the VPC to allow the ELB facing to the internet, or place the backend sytems in a private-facing subnet with no Internet access.
This [link](https://aws.amazon.com/vpc/) provide more details

### Security group

A security group is a set of firewall rules that control the traffic for any amazon services (ELB, Cluster, Service, Amazon RDS) that it ties to. For example, when create an amazon elastic load balancer

This [link](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html) provide more details

## We will create all the items in the above diagram

In this demostration, our todo API will run insidie a the default VPC in N.Virginia region (us-east-1).

We are going the create: Amazon ELB, ECS Cluster, Amazon RDS - PostgreSQL, Cluster service and Task. All of these services will run into a default VPC, and we will assign a security group for each of these service to help these service can communicate with each others more secure. Suppose we use security group on Amazon RDS to help the PostgreSQL database only accept connection from the Cluster service.
In this demo we create the following security groups:

| Security Group     |      Amazon services           |
|------------------  |:------------------------------:|
| flask-todo-elb-sg  | ELB - Elastic Load Balancer    |
| flask-todo-db-sg   | Amazon RDS - PostgeSQL         |
| flask-todo-ecs-sg  | Service inside the ECS Cluster |

I. [Security Group](#sg)

II. [Amazon ELB](#elb)

III. [ECS Cluster](#ecs)

IV. [Amazon RDS - PostgreSQL](#rds)

V. [Task](#task)

VI. [Service](#service)

## I. <a name="sg"></a>Security group

1. flask-todo-elb-sg: Security group for Elastic Load Balancer
* Group name: flask-todo-elb-sg
* Description: A security group for ELB
* VPC: Select default VPC
* Inbound Rules

  | Type     |  Protocol  |  Port Range  |  Source  |
  |--------- |:----------:|:------------:|----------|
  | HTTP (80)| TCP (6)    | 80           |0.0.0.0/0 |
* Outbound Rules

  | Type        |  Protocol  |  Port Range  |  Destination  |
  |-------------|:----------:|:------------:|---------------|
  | All traffic | All        | All          |   0.0.0.0/0   |

2. flask-todo-ecs-sg: Security group for ECS Cluster service
* Group name: flask-todo-ecs-sg
* Description: Security group for ECS Service in fargate
* VPC: Select default VPC
* Inbound Rules

  | Type            |  Protocol  |  Port Range  |  Source               |
  |-----------------|:----------:|:------------:|-----------------------|
  | Custom TCP Rule | TCP (6)    | 5000         |Pick flask-todo-elb-sg |

  Select source = flask-todo-elb-sg to allow only connection from ELB to ECS Cluster service on port 5000.
* Outbound Rules

  | Type        |  Protocol  |  Port Range  |  Destination  |
  |-------------|:----------:|:------------:|---------------|
  | All traffic | All        | All          |   0.0.0.0/0   |

3. flask-todo-db-sg: Security group for Amazon RDS - PostgeSQL
* Group name: flask-todo-db-sg
* Description: Security group for Database
* VPC: Select default VPC
* Inbound Rules

  | Type              |  Protocol  |  Port Range  |  Source               |
  |-------------------|:----------:|:------------:|-----------------------|
  | PostgreSQL (5432) | TCP (6)    | 5432         |Pick flask-todo-ecs-sg |

  Select source = flask-todo-ecs-sg to allow only connection from ECS Cluster service to the DB on port 5432.
* Outbound Rules

  | Type        |  Protocol  |  Port Range  |  Destination  |
  |-------------|:----------:|:------------:|---------------|
  | All traffic | All        | All          |   0.0.0.0/0   |

## II. <a name="elb"></a>Amazon ELB

### 1. Create an Application Load Balance

Go to this link to start create an application load balancer: [Link](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#LoadBalancers:sort=loadBalancerName)

### 2: Configure Load Balancer

![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/elb-configure-basic1.png)

![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/elb-configure-basic2.png)

### 3: Configure Security Setting

TBD

### 4: Configure Security Groups

![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/elb-configure-security-group.png)

### 5: Configure Routing

![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/elb-configure-routing.png)

To discover the availability of your services, a load balancer periodically sends pings, attempts connections, or sends requests to test to the services in ECS Cluster. These tests are called health checks. The status of the service that are healthy at the time of the health check is InService. The status of any services that are unhealthy at the time of the health check is OutOfService. The load balancer performs health checks on all registered ECS cluster services, whether the service is in a healthy state or an unhealthy state.

The load balancer routes requests only to the healthy services. When the load balancer determines that an service is unhealthy, it stops routing requests to that service. The load balancer resumes routing requests to the service when it has been restored to a healthy state.
[ELB Healchecks](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-healthchecks.html)

### 6. Register Targets

Ingore this steps we will register services tie to this ELB when creating service in cluster.

Click Review then Create button to create the ELB - Application Load Balancer.

## III. <a name="ecs"></a>ECS Cluster

[ECS CLuster](https://console.aws.amazon.com/ecs/home?region=us-east-1#/clusters)

Create a cluster powered by Fargate. We just need to input the cluster name.

## IV. <a name="rds"></a>Amazon RDS - PostgreSQL

Access this [link](https://console.aws.amazon.com/rds/home?region=us-east-1#) to start.

* Create database
  * Select engine:
    * select PostgreSQL
  * Choose use case
    * Select Dev/Test. We dont need Production for this demo.
  * Specific DB details
    * Choose the DB engine version you want. I pick the latest one.
    * DB instance class
      * I pick the db.t2.micro
      * Dont need Multi-AZ
    * Settings
      * DB instance identifier
      * Master username and password
  * Configure advanced settings
    * Pick the default VPC
    * Default for subnet group
    * Public accessibility: No. Our database aren't facing to the internet.
    * Select the VPC security group for our database. I pick the flask-todo-db-sg. We already config this security group to allow connection from flask-todo-ecs-sg so our todo API service can connect to this database.
  * Database options:
    * Input database name.
    * Leave others config as default.

  Click "Create databse"

## V. <a name="task"></a>Create Task definition

![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/task-definition.png)

* Create a Fargate task with the following information:
  * Task Definition Name: {your task name}
    * Task Role: None
    * Network: awsvpc
    * Task execution role: ecsTaskExecutionRole
    * Task memory: The total memory of running containers in this task. My container is small then I pick 0.5 GB (the smallest memory in the dropdown.)
    * Task CPU: 0.25 vCPU

* Create container
  * ![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/ecs-container-1.png)

  * ![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/ecs-container-2.png)

  * ![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/ecs-container-3.png)

  * Container Name: {your container name}
  * Image: {Link to our docker image reposistory}. TODO: Update private image security.
  * Memory limit - Hard limit: 500 MB. If we defined this value and the container attempts to exceed the hard limit, the container is killed. This field is optional with ECS Fargate.
  * Memory limit - Soft limit: 300 MB. The soft limit (in MiB) of memory to reserve for the container
    For example, if your container normally uses 128 MiB of memory, but occasionally bursts to 256 MiB of memory for short periods of time, you can set a memoryReservation of 128 MiB, and a memory hard limit of 300 MiB. This configuration would allow the container to only reserve 128 MiB of memory from the remaining resources on the container instance, but also allow the container to consume more memory resources when needed.
  * Container Port
        The open port on our container. My flask application use port 5000 then I input port 5000 here.

  * Advanced container configuration
    * HealCheck: Ignore it. We can config in ELB - Application load balancer.
    * Environment
      * CPU Units: The number of cpu units to reserve for the container. The total CPU Units for all containers in a task must be less than the CPU Units we define in the Task CPU above.
      * Essentials: True. We are creating the only one flask application container on this task then we mark this container is True to stop the task if this container is failure.
      * Entry point: The command to start our flask application. I use bin/flask-run.sh
      * Command: Leave it blank because we already define the entry point. Please check out this link to understand entry point, command and their best practice recommendation. [Link](https://medium.freecodecamp.org/docker-entrypoint-cmd-dockerfile-best-practices-abc591c30e21)
      * Working directory
        * The default working directory in the container. Leave it blank if we would like to use the default directory defined in the the Dockerfile.
      * Env Variables The environment variables such as: Database login credential, other configs:
        * DB_HOST: Database Host Name
        * DB_NAME: Database Name
        * DB_PWD: Database Password
        * DB_USER_NAME: Database User name
        * FLASK_APP: The flask application variable environment.
        * FLASK_ENV: production
        * PYTHONPATH: /app/src/ This folder contain all modules in todo application. We define to help python find out our modules.
      * Networking: Ignore it.
      * Storage and logging: Leave everything as default.
      * Resource limits and Docker Label: Ignore it.

## VI. <a name="service"></a>Create service to run the above task

### 1. Create service

    Name: flask-todo-service

#### 1. Configure service

![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/ecs-configure-service.png)

#### 2. Configure network

Security group: flask-todo-ecs-sg

![DNS](https://raw.githubusercontent.com/TranHoang/flask-helloworld-devops/master/aws/images/ecs-service-configure-network.png)

#### 3. Set auto scaling

Use default. I will update the autoscaling configuration.

We are all set. Please go the application load balancer to get the public domain name then try to access our api.

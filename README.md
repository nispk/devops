# Deploy HTML website on AWS EC2 instance

In this project, we deploy an HTML website on AWS EC2 instance using DevOps process. The tools that are used for this DevOps process are AWS, Jenkins and Docker. We build a CI/CD pipeline on Jenkins where we run 3 jobs to deploy an HTML website on EC2 instance. The devops process flow is as shown below:

![DevOps process](DevOps_process.jpg?raw=True)

## Steps followed

### 1) Setup EC2 instance

Generally EC2 instance can be launched from the EC2 dashboard by choosing the type of AMI (Amazon Machine Image) Linux, MacOs or Windows etc, configuring the network settings
and assigning a key pair for remotely accessing the ec2 instance via SSH client or putty. We are using a free tier AWS account that allows users upto 30 GB of free storage. In order to ensure successfull access to an EC2 instance remotely or outbound traffic from EC2 instance, we need to fulfill some pre-requisites such as-
  1. Setup Virtual Private Cloud(VPC)
  2. Setup subnets, route tables and gateways
  3. Setup KeyPair 
  4. Setup Security groups

**Setup Virtual Private Cloud**

An AWS Virtual Private Cloud is a private virtual network where users can host servers, data´centers or anyother computing system that can be remotely accessed. We can create an individual VPC or VPC along with other resources which will be helpful when we launch EC2 instances and remotely access them. In our case, we have created a VPC that also creates new subnets, route tables and gateways within that VPC. For our VPC we specify an IPv4 CIDR block of `10.0.0.0/16` which gives us 65,536 ip addresses




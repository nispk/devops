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

An AWS Virtual Private Cloud is a private virtual network where users can host servers, data´centers or anyother computing system that can be remotely accessed. We can create an individual VPC or VPC along with other resources which will be helpful when we launch EC2 instances and remotely access them. In our case, we have created a VPC that also creates new subnets, route tables and gateways within that VPC. For our VPC we specify an IPv4 CIDR block of `10.0.0.0/16` which gives us 65,536 ip addresses, choose 2 availability zones for high availability in case one of the zones is down, create 2 public subnets and 0 NAT gateways since we will not be using private subnet for our EC2 instance.

**Setp subnets, route tables and gateways**

In the previous step, the subnets, route tables and gateways are created by default within the new VPC. However we need to configure few details for the subnets. Since we need to deploy a web application on our EC2 instance which can be accessed from the internet, we have created only public subnet with IPv4 CIDR block `10.0.16.0/20` so that we can have 4096 ip addresses. The CIDR block for subnets depends on the VPC CIDR block, we can have as many subnets depending on the range of ip addresses. For a VPC of `/16` CIDR mask we get 65,536 ip addresses which allows us to define upto 16 subnets with CIDR mask of `/20` for each subnet. We either auto-assign a public IPv4 address while creation 
or we can allocate a new elastic IP address and associate it to the public subnet. 

When a VPC is created (as in the previous step) automatically a main route table is created without any explicit subnet assocciations. The main route table has only one entry with the destination as VPC CIDR block and it's target being local which allows the traffic between the VPC resources. To enable a public subnet to send outbound traffic to internet a custom route table is created by the VPC. This route table has 2 entries. One for the local communication and the other with the target as internet gateway, which sends the outbound traffic from both the public subnets to the internet gateway. Hence allowing internet access for the subnet. In this method, the public subnets are explicitly assocciated with the custom route table.



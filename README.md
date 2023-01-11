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

When a VPC is created (as in the previous step) automatically a main route table is created without any explicit subnet assocciations. The main route table has only one entry with the destination as VPC CIDR block and it's target being local which allows the traffic between the VPC resources. To enable a public subnet to send outbound traffic to internet a custom route table is created by the VPC. This route table has 2 entries. One for the local communication and the other with the target as internet gateway, which sends the outbound traffic from both the public subnets to the internet gateway. Hence allowing internet access for the subnet through the gateway. In this method, the public subnets are explicitly assocciated with the custom route table.

The internet gateways enables the resources in public subnet to connect to internet if it has a public ip address at the same time enabling remote access from the internet to the resources in the public subnet. For an outbound traffic sent from a public subnet, the internet gateways set the reply ip address as the public ip address of the subnet, so that when an inbound traffic is recieved from the internet the internet gateway translates the destination of the traffic to the private ip address of the resource in the public subnet. Hence for successfull communication between the internet and the AWS resources in a VPC, we create an internet gateway and attach it to the VPC. Finally we enter the internet gateway into the custom route table for `0.0.0.0/0` as the destination. If the VPC is created as per the previous step, then automatically the internet gateway is also created, attached to the VPC and added as an entry in the custom route table.

**Setup Keypair**

An EC2 instance requires a Key Pair of public key and private key for authentication while connecting to an SSH client or putty. We have created an rsa key pair of .pem type since we have an SSH client. For windows users putty can be used to connect to an EC2 instance which requires rsa key pair of .ppk type. While connecting to EC2 instance via putty or SSH client it is required to store the public key with correct permissions to form a successfull connection. 

**Setup Security Groups**

Security groups are necessary to filter the traffic that is allowed to reach and leave the resources that it is assocciated with. In our case, we setup security groups for our EC2 instances so that we can control who is allowed to send the traffic to our EC2 instances and where it can go from the EC2 instances. We have set inbound rules as -
  1. Allow `TCP protocol` traffic at `Port 8080` from Source `0.0.0.0/0` . This is for the inbound traffic from main node of the jenkins server to the EC2 instance.
  2. Allow `TCP protocol` traffic at `Port 82` from Source `0.0.0.0/0`. This is for the inbound traffic from the Docker container to the slave node in jenkins installed at EC2 instance.
  3. Allow `TCP protocol` traffic at `Port 80` from Source `0.0.0.0/0`. This is for the inbound traffic from the Docker container to the slave node in jenkins installed at the other EC2 instance. 
  4. Allow `TCP protocol` traffic at `Port 43427` from Source `0.0.0.0/0`. This is for the inbound traffic from the slave nodes in the jenkins server to the EC2 instance.
  5. Allow `SSH protocol` traffic at `Port 22` from Source `0.0.0.0/0`. This is for the inbound traffic from our local computer to the EC2 instances.
 
 Apart from the above inbound traffic rules one traffic rule which allows all types of traffic from any source is added by default when a new VPC is created. We can filter the incoming traffic on the source level as well by specifying a single ip address for eg: local computer or server ip address instead of general source `0.0.0.0/0`. For the outbound rules we have set all type of traffic from all ports towards any destination, which allows the EC2 instances to communicate with any server, computer etc on internet.
 
### 2) Setup Jenkins

Since we will use Jenkins to build our CI/CD pipeline, We install jenkins on 3 Ec2 instances named- Jenkins main server, staging server and production server. We will be using Jenkins main server instance to setup our pipeline, the staging server instance for testing and production server to see the final results of our deployment. In order to setup the jenkins server we follow below steps-
  1. Check if all the softwares are updated and install java 
      ```
      sudo apt update
      sudo apt install openjdk-11-jdk -y
      ```
  2. Add jenkins repository by first importing the GPG key. The GPG key verifies package integrity.
      ```
      curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | 
      sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
      ```
      Now we add the jenkins software repository to the source list and provide the authentication key.
      ```
      echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | 
      sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
      ```
  3. Install jenkins
     ```
     sudo apt update
     sudo apt install jenkins -y
     ```
  4. Check if the jenkins is running if not activate.
     ```
     sudo systemctl status jenkins
     sudo systemctl enable now --jenkins
     ```
  5. Launch the jenkins server with the ip address of the EC2 instance on port 8080. Once we launch the jenkins, we setup the user account, check if the GitHub plugin and the Credentials Plugin exist, if not install these plugins. For the built-in node configure the `No. of executers = 2` and `Ùsage = Use this node as much as possible`.
  6. Create 2 more nodes and configure them. First node called as staging and second node called as production. Configure both the servers with `No. of executers = 1`, `Remote Root directory = /home/ubuntu/jenkins` , `Launch method = Launch agent by connecting it to the controller`, `Internal data directory = remoting`, `Availability = Keep this agent online as much as possible`.
  7. To connect both the slave nodes to the EC2 instances, we run the commands for each agent on the nodes on each EC2 instance command line. 
      
     

### 3) Setup Docker container

### 4) Setup CI/CD Pipeline

  

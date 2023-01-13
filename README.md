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

An AWS Virtual Private Cloud is a private virtual network where users can host servers, data´centers or any other computing system that can be remotely accessed. We can create an individual VPC or VPC along with other resources which will be helpful when we launch EC2 instances and remotely access them. In our case, we have created a VPC that also creates new subnets, route tables and gateways within that VPC. For our VPC we specify an IPv4 CIDR block of `10.0.0.0/16` which gives us 65,536 ip addresses, choose 2 availability zones for high availability in case one of the zones is down, create 2 public subnets and 0 NAT gateways since we will not be using private subnet for our EC2 instance.

**Setp subnets, route tables and gateways**

In the previous step, the subnets, route tables and gateways are created by default within the new VPC. However we need to configure few details for the subnets. Since we need to deploy a web application on our EC2 instance which can be accessed from the internet, we have created only public subnet with IPv4 CIDR block `10.0.16.0/20` so that we can have 4096 ip addresses. The CIDR block for subnets depends on the VPC CIDR block, we can have as many subnets depending on the range of ip addresses. For a VPC of `/16` CIDR mask we get 65,536 ip addresses which allows us to define upto 16 subnets with CIDR mask of `/20` for each subnet. We either auto-assign a public IPv4 address while creation 
or we can allocate a new elastic IP address and associate it to the public subnet. 

When a VPC is created (as in the previous step) automatically a main route table is created without any explicit subnet assocciations. The main route table has only one entry with the destination as VPC CIDR block and it's target being local which allows the traffic between the VPC resources. To enable a public subnet to send outbound traffic to internet a custom route table is created by the VPC. This route table has 2 entries. One for the local communication and the other with the target as internet gateway, which sends the outbound traffic from both the public subnets to the internet gateway. Hence allowing internet access for the subnet through the gateway. In this method, the public subnets are explicitly assocciated with the custom route table.

The internet gateways enables the resources in public subnet to connect to internet if it has a public ip address at the same time enabling remote access from the internet to the resources in the public subnet. For an outbound traffic sent from a public subnet, the internet gateways set the reply ip address as the public ip address of the subnet, so that when an inbound traffic is recieved from the internet the internet gateway translates the destination of the traffic to the private ip address of the resource in the public subnet. Hence for successfull communication between the internet and the AWS resources in a VPC, we create an internet gateway and attach it to the VPC. Finally we enter the internet gateway into the custom route table for `0.0.0.0/0` as the destination. If the VPC is created as per the previous step, then automatically the internet gateway is also created, attached to the VPC and added as an entry in the custom route table.

**Setup Keypair**

An EC2 instance requires a Key Pair of public key and private key for authentication while connecting to an SSH client or putty. We have created an rsa key pair of .pem type since we use an SSH client. For windows users putty can be used to connect to an EC2 instance which requires rsa key pair of .ppk type. While connecting to EC2 instance via putty or SSH client it is required to store the public key with correct permissions to form a successfull connection. 

**Setup Security Groups**

Security groups are necessary to filter the traffic that is allowed to reach and leave the resources that it is assocciated with. In our case, we setup security groups for our EC2 instances so that we can control who is allowed to send the traffic to our EC2 instances and where it can go from the EC2 instances. We have set inbound rules as -
  1. Allow `TCP protocol` traffic at `Port 8080` from Source `0.0.0.0/0` . This is for the inbound traffic from main node of the jenkins server to the EC2 instance.
  2. Allow `TCP protocol` traffic at `Port 82` from Source `0.0.0.0/0`. This is for the inbound traffic from the Docker container to the slave node in jenkins installed at EC2 instance.
  3. Allow `TCP protocol` traffic at `Port 80` from Source `0.0.0.0/0`. This is for the inbound traffic from the Docker container to the slave node in jenkins installed at the other EC2 instance. 
  4. Allow `TCP protocol` traffic at `Port 43427` from Source `0.0.0.0/0`. This is for the inbound traffic from the slave nodes in the jenkins server to the EC2 instance.
  5. Allow `SSH protocol` traffic at `Port 22` from Source `0.0.0.0/0`. This is for the inbound traffic from our local computer to the EC2 instances.
 
 Apart from the above inbound traffic rules one traffic rule which allows all types of traffic from any source is added by default when a new VPC is created. We can filter the incoming traffic on the source level as well by specifying a single ip address for eg: local computer or server ip address instead of general source `0.0.0.0/0`. For the outbound rules we have set all type of traffic from all ports towards any destination, which allows the EC2 instances to communicate with any server, computer etc on internet.
 
### 2) Setup Jenkins

Since we will use Jenkins to build our CI/CD pipeline, We install jenkins on 3 EC2 instances named- Jenkins main server, staging server and production server. We will be using Jenkins main server instance to setup our pipeline, the staging server instance for testing and production server to see the final results of our deployment. In order to setup the jenkins server we follow below steps-
  1. Check if all the softwares are updated and install java on all instances
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
  3. Install jenkins on all instances
     ```
     sudo apt update
     sudo apt install jenkins -y
     ```
  4. Check if jenkins is running if not activate on one instance which acts as the main node in our automation.
     ```
     sudo systemctl status jenkins
     sudo systemctl enable now --jenkins
     ```
  5. Launch the jenkins server with the ip address of the EC2 instance on port 8080. Once we launch the jenkins, we setup the user account, check if the GitHub plugin and the Credentials Plugin exist, if not install these plugins. For the built-in node configure the `No. of executers = 2` and `Ùsage = Use this node as much as possible`. Configure Global security with `Security Realm = Jenkins own user database`, `Authorization = Logged-in users can do anything`, `TCP port for inbound agents = Random`, `API Token = Enable API token usage statistics`, `SSH server = Disable`. Save the settings.
  6. Create 2 more nodes and configure them. First node called as staging and second node called as production. Configure both the servers with `No. of executers = 1`, `Remote Root directory = /home/ubuntu/jenkins` , `Launch method = Launch agent by connecting it to the controller`, `Internal data directory = remoting`, `Availability = Keep this agent online as much as possible`. Save the settings.
  7. To connect both the slave nodes to the EC2 instances, we run the commands for the agents on each EC2 instance command line. 

### 3) Setup Docker container

Docker containers are very useful to run a web application making the process really simple by keeping all the dependencies required to build it in one place and easily deploy it on any OS, giving the application the highest abstraction. In our case, we have to deploy a simple website, hence we need a web server to process, store and manage the HTTPS requests from the client system (end user) and display it over the internet. For this purpose we have used a Docker image of Apache container. The Apache container acts as the base image on top of which we develop our docker container in which we run the website. The steps followed-
  1. We first install docker in the EC2 instances where slave nodes are present - `sudo apt-get install docker.io` 
  2. To build our Apache container that can be used as the base image for our website container, we have created a github package that contains the Apache docker image.
  3. We build our final docker image on top of the Apache container with the help of Docker file that hosts our website.
  
  **Setup Github Package** 
  
  The Github package is stored in Github Container repository, that can be used as docker container. To build a github package, we need 2 input files, 
  
  1. the [YAML file](https://github.com/nispk/webapps/blob/main/.github/workflows/publish.yml) 
  2. the [Docker file](https://github.com/nispk/webapps/blob/main/Dockerfile) 

  Github Actions is an automation platform that can be used to build, test and push images to the Github Container repository similar to Docker hub. The yaml file is the github actions workflow file that contains the necessary information to build our github package such as the jobs, the OS on which the jobs are run, the steps for each job. 
To build a docker image, we use docker github action commands that allow us to login to the Github container repository using the username and the personal access token of the Github repository and run the commands in the Docker file of the repository and build the image. The github actions workflow is triggered whenever there is a push on the 'main' branch. The name of the docker image is stored in the format of `ghcr.io/{Github username}/conatiner_name:version`. Once the docker image is pushed to the repository we can pull it with the command `docker pull ghcr.io/{Github username}/conatiner_name:version` and run it in a container similar to docker.

### 4) Setup CI/CD Pipeline

For our DevOps process we have 3 jobs in the CI/CD pipeline. Since our goal is to deploy a website on the AWS EC2 instance. We have created 3 jobs-
1. Git job
2. Test job
3. Push to Prod job

**Git job:**

This job is run on the staging server. This job is part of CI step in the pipeline where we integrate the code into the system. Here we are cloning the files from the Github repository to the staging server. (**Note**: Since the github repository is a public repository we do not need to store credentials in jenkins, else we have to store the username and personal access token of the repository in the credentials manager of jenkins and use these credentials in the source code management, when we add the repository) 

In order for the pipeline to be triggered everytime there is any change in the source code (push to the repository), we add a webhook to our Github repository linking the jenkins server to the repository. Similarly in the jenkins server, we configure our job to be built if there is a trigger from GitHub hook trigger for GITScm polling. We add the next job to the pipeline in the Post-build actions of this job, which basically triggers the **Test job** once the Git job is run successfully.

**Test job:**

This job is also run on the staging server. We want to test run our code that was cloned in the previous step on the staging server instance before deploying it on the production server. This job will trigger the docker container to run with the below commands in shell. Hence deploying the website on the staging server instance. We can check if the website is deployed successfully in the browser through the `https://{ip address of the EC2 instance}:82`. 

```
sudo docker rm -f $(sudo docker ps -a -q)
sudo docker build /home/ubuntu/jenkins/workspace/Git-job/ -t website
sudo docker run -it -p 82:80 -d website
```
If this job runs succesfully, the next job in the pipeline which is **Push to Prod job** is triggered.

**Push to Prod job:**

This job is run on the production server. We clone the github repository on the production server and later run the below commands in the shell. We can check if the website is deployed successfully in the browser through the `https://{ip address of the EC2 instance}:80`. 

```
sudo docker rm -f $(sudo docker ps -a -q)
sudo docker build /home/ubuntu/jenkins/workspace/push-prod/ -t website
sudo docker run -it -p 80:80 -d website
```

## Conclusion

This completes the deployement of our CI/CD pipeline. We see how the Docker,Jenkins and Github play a major role in completing the pipeline and also the AWS EC2 instances provide us with the necessary resources to see our results. 

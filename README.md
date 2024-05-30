<h2 align="center"> ACCESS INTO EC2 INSTANCES WITHOUT A BASTION HOST</h2>

In this project, the task was to create two (2) private subnets, (Private Subnet A and Private Subnet B), and two (2) public subnets, (Public Subnet A and Public Subnet B), in two different Availability Zones (AZ), (this to enhance availability and fault tolerance). Using Auto Scaling Group, create EC2 instances in the Private Subnets, the EC2 Instances would communicate with the internet using NAT Gateway created in each of the AZ in the public subnets. Access should also be gained into one of the EC2 instances to manually increase CPU utilization (to simulate high network traffic and usage) so that the Auto Scaling Group can respond to the CPU usage and create additional instances to cater for the high request volume.

Below is the Architecture Diagram.

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/network-architecture.png" height="60%" width="75%"/>
</p>
<h5 align="center"> Network Architecture Diagram</h5>

<h2 align="left"> Gaining Access into the EC2 Instance(s) </h2>

One way of gaining access to EC2 instance(s) in a private network without a Bastion Host is by the use of a System Manager (SSM). This method allows managing and accessing EC2 instances securely without exposing them to the internet or needing a bastion host, leveraging AWS's built-in tools and policies for secure instance management. To get this working, I carried out the following steps:
<h5 align="left">
✔️ created an IAM Role (SSMRole) and attached a trust policy document to it (JSON file). 
</h5>
<h5 align="left">
✔️ created an IAM policy (SSMPolicy) using the AmazonSSMManagedInstanceCore template on the AWS console.
</h5>
<h5 align="left">
✔️attached the IAM policy (SSMPolicy) to the IAM role (SSMRole)
</h5>
<h5 align="left">
✔️created an IAM Instance Profile (SSMInstanceProfile). (This is what I attached to the launch template, to gain access to the EC2 instance(s) created by the Auto Scaling Group)
</h5>
<h5 align="left">
✔️ attached the IAM Role (SSMRole) to the IAM Instance Profile (SSMInstanceProfile) before attaching it to a launch template.
</h5>
<h5 align="left">
✔️ after the EC2 instances have been launched by ASG, session manager is then used to gain access to the instance(s)
</h5>
<h5 align="left">
I used AWSCLI for this section.
</h5>
<h5 align="left">
trustpolicy.json
</h5>

```commandline
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

AmazonSSMManagedInstanceCore.json
```commandline
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:DescribeAssociation",
                "ssm:GetDeployablePatchSnapshotForInstance",
                "ssm:GetDocument",
                "ssm:DescribeDocument",
                "ssm:GetManifest",
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:ListAssociations",
                "ssm:ListInstanceAssociations",
                "ssm:PutInventory",
                "ssm:PutComplianceItems",
                "ssm:PutConfigurePackageResult",
                "ssm:UpdateAssociationStatus",
                "ssm:UpdateInstanceAssociationStatus",
                "ssm:UpdateInstanceInformation"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssmmessages:CreateControlChannel",
                "ssmmessages:CreateDataChannel",
                "ssmmessages:OpenControlChannel",
                "ssmmessages:OpenDataChannel"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2messages:AcknowledgeMessage",
                "ec2messages:DeleteMessage",
                "ec2messages:FailMessage",
                "ec2messages:GetEndpoint",
                "ec2messages:GetMessages",
                "ec2messages:SendReply"
            ],
            "Resource": "*"
        }
    ]
}
```
CREATING SSM ROLE

```commandline
aws iam create-role --role-name SSMRole --assume-role-policy-document file://"C:/Users/Tony Pc/Downloads/trustpolicy.json"
```
<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/ssm-role.png" height="55%" width="65%"/>
</p>

ATTACHING AMAZON SSM MANAGED INSTANCE CORE TO SSM ROLE

I didn't have to create the Amazon SSM managed instance core because it already exists on the AWS console, so I just attached the policy to the SSMRole.

```commandline
aws iam attach-role-policy --role-name SSMRole --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

CREATING IAM INSTANCE PROFILE

```commandline
aws iam create-instance-profile --instance-profile-name SSMInstanceProfile
```

ADDING IAM ROLE TO INSTANCE PROFILE

```commandline
aws iam add-role-to-instance-profile --instance-profile-name SSMInstanceProfile --role-name SSMRole
```

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/iam-instance-profile.png" height="40%" width="60%"/>
</p>


<h2 align="left"> Creating VPC</h2>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/vpc.png" height="60%" width="60%"/>
</p>

<h2 align="left"> Creating IGW</h2>

Next, I created the internet gateway which I attached to the vpc.

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/igw.png" height="50%" width="65%"/>
</p>

<h2 align="left"> Subnet Creation</h2>
I created four subnets, a private and public subnet in one availability zone (us-east-1a) and another set in another availability zone (us-east-1c). This increases availability and fault tolerance. 

<p>
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/public-subnet-a.png" height="50%" width="45%"/>
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/public-subnet-b.png" height="50%" width="45%"/>
</p>

<p>
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/private-subnet-a.png" height="50%" width="45%"/>
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/private-subnet-b.png" height="50%" width="45%"/>
</p>

<h2 align="left"> Creating NAT Gateways</h2>

The next step is the creation of NAT gateways in each of the availability zones in the public subnets. It is through the NAT gateway that the EC2 instances in each of the AZs would communicate with the internet. Also, providing a NAT gateway in each AZ ensures availability and possibly no downtime.

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/nat-gw-a1.png" height="60%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/nat-gw-a2.png" height="40%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/nat-gw-b1.png" height="60%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/nat-gw-b2.png" height="55%" width="65%"/>
</p>


<h2 align="left"> Creating Route Tables</h2>

I created two route tables aside from the one that was created by the VPC. The route table created by the VPC was renamed "Main-RT", which routed all internet-bound traffic on the public subnet through the Internet Gateway, then a routeing table for each of the subnets, to route internet-bound traffic from each private subnet in each availability zone through the associated NAT Gateway in the public subnet.

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/private-subnetA-rt.png" height="60%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/private-subnetB-rt.png" height="60%" width="60%"/>
</p>

<h2 align="left"> Editing Routes and Associating Subnets</h2>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/editing-route-main-rt.png" height="70%" width="85%"/>
</p>
<h5 align="center"> Editing Route through IGW for Main RT</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/editing-route-private-subnetA-rt.png" height="70%" width="85%"/>
</p>
<h5 align="center"> Editing Route through "Subnet-A-NAT-GW" for "Private-SubnetA-RT</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/associating-subnet-A-to-private-subnetA-rt.png" height="70%" width="85%"/>
</p>
<h5 align="center"> Associating Subnet A to "Private-SubnetA-RT"</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/editing-route-private-subnetB-rt.png" height="70%" width="85%"/>
</p>
<h5 align="center"> Editing Route through "Subnet-B-NAT-GW" for "Private-SubnetB-RT</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/associating-subnet-B-to-private-subnetB-rt.png" height="70%" width="85%"/>
</p>
<h5 align="center"> Associating Subnet B to  "Private-SubnetB-RT"</h5>

<h2 align="left"> Security Group Configuration</h2>

The next step is the creation and configuration of the security group. I created two security groups, one for the Application Load Balancer (ALB-SG) and the other for the private subnet (Private-SG). The ALB-SG allows only HTTP and HTTPS traffic from the internet to reach the ALB (Inbound rule) while HTTP and HTTPS traffic from the ALB are sent to the Private-SG (Outbound rule). The Private SG only allows HTTP and HTTPS traffic from the ALB to reach the Private Subnet (Inbound rule). 
The purpose of this configuration is to effectively use security groups to manage and restrict traffic flow, ensuring that only legitimate traffic reaches the EC2 instances in the private subnets while protecting them from direct exposure to the internet.

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/sg-overview.png" height="65%" width="75%"/>
</p>
<h5 align="center"> Overview of the Security Groups Created</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/editing-inbound-rule-for-alb-sg.png" height="65%" width="75%"/>
</p>
<h5 align="center"> Inbound Rules for ALB-SG</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/editing-outbound-rules-for-alb-sg.png" height="65%" width="75%"/>
</p>
<h5 align="center"> Outbound Rules for ALB-SG</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/editing-inbound-rule-for-private-sg.png" height="65%" width="75%"/>
</p>
<h5 align="center"> Inbound Rules for Private-SG</h5>


<h2 align="left"> Creating a Launch Template</h2>

I did two major configurations, aside from the general configurations. First I attached the IAM instance profile I created using the AWS CLI to the launch template. This allows me to securely connect to the EC2 instances in the private subnets using session manager. The other is that I created and attached a user data script that installs stress and Apache2, enables and starts Apache2, sets the timezone to GMT+1 for Africa/Nigeria, gets the current time in the time zone and writes the instance's IP address, date and time the EC2 instance was created into an html file (index.html) and stored in /var/www/html directory. This is used for health checks and displays the IP address and time the EC2 instance was created. This helps to validate that the ALB is working.

user data
```commandline
#!/bin/bash
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get install -y stress apache2
sudo systemctl enable apache2
sudo systemctl start apache2
sudo timedatectl set-timezone Africa/Lagos
current_time=$(date)
cd /var/www/html
echo "This is INSTANCE $(hostname). Created on ${current_time}" | sudo tee index.html
```
<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/launch-template1.png" height="55%" width="48%"/>
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/launch-template2.png" height="55%" width="48%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/launch-template3.png" height="55%" width="48%"/>
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/launch-template4.png" height="55%" width="48%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/launch-template5.png" height="65%" width="75%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/launch-template6.png" height="65%" width="75%"/>
</p>

<h2 align="left"> Creating Target Group</h2>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/target-group1.png" height="55%" width="65%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/target-group2.png" height="55%" width="65%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/target-group3.png" height="55%" width="65%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/target-group4.png" height="55%" width="65%"/>
</p>

No EC2 instances were added yet

<h2 align="left"> Creating an Application Load Balancer</h2>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/alb1.png" height="60%" width="70%"/>
</p>
<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/alb2.png" height="60%" width="70%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/alb3.png" height="60%" width="70%"/>
</p>
<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/alb4.png" height="60%" width="70%"/>
</p>


<h2 align="left"> Creating Auto Scaling Group</h2>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/asg1.png" height="40%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/asg2.png" height="40%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/asg3.png" height="40%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/asg4.png" height="40%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/asg5.png" height="40%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/asg6.png" height="40%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/asg7.png" height="40%" width="60%"/>
</p>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/asg8.png" height="40%" width="60%"/>
</p>


<h2 align="left"> Validating Set-up</h2>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/first-instance-by-asg.png" height="60%" width="70%"/>
</p>
<h5 align="center"> First EC2 Instance Created by ASG</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/apach2-working.png" height="60%" width="70%"/>
</p>
<h5 align="center"> User Data for Health Check Works</h5>

CONNECTING TO EC2

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/connecting-via-system-manager.png" height="60%" width="70%"/>
</p>
<h5 align="center"> Connecting to EC2 Instance Via Session Manager</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/access-gained-into-ubuntu-server.png" height="80%" width="90%"/>
</p>
<h5 align="center"> Access Gained into EC2 Instance</h5>

After gaining access to the EC2 instance, I used the stress command to increase the CPU utilization manually. When the average CPU utilization exceeds 75%, the ASG will launch new instances. 

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/average-cpu-utilization-before-stress-command.png" height="60%" width="70%"/>
</p>
<h5 align="center"> Average CPU Utilization before the Stress Command</h5>

stress command
```commandline
stress -c 200
```

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/average-cpu-utilization-after-stress-command.png" height="60%" width="70%"/>
</p>
<h5 align="center"> Average CPU Utilization Spikes Over 75% leading to the creation of New EC2 Instances to cater for high demand</h5>

As the average CPU utilization continues to increase, and overwhelms the second EC2 instance, a third EC2 instance is launched by the Auto Scaling Group (ASG) using the launch template. The Application Load Balancer also effectively routes traffic to all the 3 EC2 instances.

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/more-instance-creation-by-asg.png" height="60%" width="70%"/>
</p>
<h5 align="center"> More EC2 Instances Created by ASG Based on Group Size Configuration in ASG</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/apach2-working.png" height="60%" width="70%"/>
</p>
<h5 align="center"> ALB Sending Traffic To First EC2 Instance</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/second-instance-created-by-asg.png" height="60%" width="70%"/>
</p>
<h5 align="center"> ALB Sending Traffic To Second EC2 Instance</h5>

<p align="center">
<img src="https://github.com/Topzdomain/Accessing-a-Private-Subnet-Without-a-Bastion-Host/blob/main/screenshots/third-instance-created-by-asg.png" height="60%" width="70%"/>
</p>
<h5 align="center"> ALB Sending Traffic To Second EC2 Instance</h5>

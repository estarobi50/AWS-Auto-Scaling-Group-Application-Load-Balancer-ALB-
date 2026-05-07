## OVERVIEW

This project demonstrates how to build a scalable AWS web application architecture using:

A Golden AMI
Launch Templates
Auto Scaling Groups (ASG)
Application Load Balancer (ALB)
Target Groups
Security Groups

The architecture uses a preconfigured EC2 instance as the “golden image,” which is cloned automatically by the Auto Scaling Group. Traffic is distributed through an Application Load Balancer across multiple EC2 instances running Apache web servers.

The goal is to validate:

Auto Scaling launches healthy EC2 instances automatically
ALB distributes traffic correctly
Instances register dynamically with the target group
Security groups properly isolate backend servers
Each web page displays unique instance identity information
## WE WILL BUILD
Base EC2 “golden instance”
Golden AMI
Launch Template
Target Group
Application Load Balancer
Auto Scaling Group
Dynamic EC2 fleet behind ALB
## FINAL ARCHITECTURE FLOW

AMI → Launch Template → Auto Scaling Group → EC2 Instances → Target Group → Application Load Balancer → Internet

## STEP-BY-STEP DEPLOYMENT
## 1. CREATE THE BASE EC2 (GOLDEN INSTANCE)

Launch a new EC2 instance.

EC2 Configuration
AMI: Amazon Linux 2
Instance Type: t2.micro
VPC: Your main VPC
Subnet: Public subnet
Public IP: Enabled
Security Group

Inbound Rules:

Type	Port	Source
HTTP	80	0.0.0.0/0
SSH (optional)	22	Your IP
## IAM ROLE (IMPORTANT)

Attach the following IAM role to the instance:

AmazonSSMManagedInstanceCore

This allows Systems Manager (SSM) access without requiring SSH.

## 2. USER DATA (WEB SERVER + INSTANCE IDENTITY)

Paste the following user data script during EC2 launch:

#!/bin/bash
exec > /var/log/user-data.log 2>&1

yum update -y
yum install -y httpd

systemctl start httpd
systemctl enable httpd

# IMDSv2 token
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)

AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

PRIVATE_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/local-ipv4)

cat <<EOF > /var/www/html/index.html
<html>
  <body>
    <h1>Web Server Active</h1>
    <h2>Instance ID: $INSTANCE_ID</h2>
    <h2>AZ: $AZ</h2>
    <h2>Private IP: $PRIVATE_IP</h2>
  </body>
</html>
EOF
## 3. VALIDATE EC2 WORKS

Connect to the instance and test locally:

curl http://localhost/

Then open a browser:

http://<public-ip>

You should see:

Instance ID
Availability Zone
Private IP

This confirms Apache and the metadata script are functioning correctly.

## 4. CREATE AMI (GOLD IMAGE)

After validation:

EC2 → Instances → Actions → Image and Templates → Create Image

Name
webserver-golden-ami

Wait until the AMI state becomes:

Available

This AMI becomes the source image for all Auto Scaling instances.

## 5. CREATE LAUNCH TEMPLATE

Navigate to:

EC2 → Launch Templates → Create Launch Template

Configuration
Setting	Value
AMI	webserver-golden-ami
Instance Type	t2.micro
IAM Role	AmazonSSMManagedInstanceCore
User Data	Leave Empty
## SECURITY GROUP FOR EC2 INSTANCES

Create or select an EC2 security group with:

Inbound Rules
Type	Port	Source
HTTP	80	ALB Security Group

Important:
Do NOT allow HTTP from 0.0.0.0/0 in the final production-style architecture.

Backend instances should only accept traffic from the ALB.

## 6. CREATE TARGET GROUP (CRITICAL STEP)

Navigate to:

EC2 → Target Groups → Create Target Group

Configuration
Setting	Value
Target Type	Instances
Protocol	HTTP
Port	80
VPC	Main VPC
Health Check Configuration
Setting	Value
Path	/
Matcher	200

This allows the ALB to determine whether instances are healthy before sending traffic.

## 7. CREATE APPLICATION LOAD BALANCER (ALB)

Navigate to:

EC2 → Load Balancers → Create Application Load Balancer

## ALB CONFIGURATION
Setting	Value
Scheme	Internet-facing
IP Type	IPv4
VPC	Main VPC
Subnets	Public Subnets
## ALB SECURITY GROUP

Inbound Rules:

Type	Port	Source
HTTP	80	0.0.0.0/0

This allows public internet traffic into the load balancer.

## LISTENER CONFIGURATION
Protocol	Port	Action
HTTP	80	Forward to Target Group
## 8. CREATE AUTO SCALING GROUP (ASG)

Navigate to:

EC2 → Auto Scaling Groups → Create Auto Scaling Group

## STEP 1 — LAUNCH TEMPLATE

Select:

Your AMI-based launch template
## STEP 2 — NETWORK
Setting	Value
VPC	Main VPC
Subnets	Private Subnets

Best practice:
Deploy Auto Scaling instances into private subnets.

## STEP 3 — ATTACH LOAD BALANCER

Choose:

Attach to existing target group

Select:

Your HTTP target group

This is the step many people forget.

Without this attachment:

Instances launch
But ALB never routes traffic to them
## STEP 4 — HEALTH CHECKS
Setting	Value
Health Check Type	ELB
Grace Period	120 seconds
## STEP 5 — CAPACITY
Setting	Value
Desired Capacity	2
Minimum Capacity	2
Maximum Capacity	4

This ensures:

Two servers always remain online
ASG can scale up during demand spikes
## 9. FIX SECURITY GROUPS (VERY IMPORTANT)
## ALB SECURITY GROUP

Inbound:

Type	Port	Source
HTTP	80	0.0.0.0/0
## EC2 INSTANCE SECURITY GROUP

Inbound:

Type	Port	Source
HTTP	80	ALB Security Group

This prevents direct internet access to backend servers.

## 10. VALIDATE THE SYSTEM
## CHECK TARGET GROUP

Navigate to:

EC2 → Target Groups → Targets

Verify:

Targets show Healthy
## TEST LOAD BALANCER

Copy the ALB DNS name and open it in a browser.

Example:

http://your-alb-dns-name

Refresh multiple times.

You should observe:

Different Instance IDs
Different private IPs
Different Availability Zones

This confirms:

Load balancing works
Multiple EC2 instances are active
Traffic is distributed correctly

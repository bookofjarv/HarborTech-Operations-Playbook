# Week 4: EC2 Evidence Lab

## HarborTech Ticket Summary

* **Ticket ID:** TKT-2026-0004
* **Client:** Riverside Goods
* **Assigned Engineer:** Jarvis D. Anderson (`xxxxxxxxxx=Jarvis_D._Anderson`)
* **AWS Account ID:** `xxxxxxxxxxx`
* **Target System:** Web Application Host running Apache (`http`) on Amazon Linux 2023
* **Reported Issue:** External client HTTP requests to the web server fail with connection timeouts.

---

## Client Impact

Riverside Goods reported that public web traffic cannot reach their web application host. The server is expected to serve standard web content over HTTP on TCP port 80. Because network attempts time out externally, customer-facing services hosted on this instance are completely unreachable, leading to reported downtime for the application.

---

## Environment and Resource Names

* **AWS Region:** `us-east-1`
* **VPC ID:** `vpc-06ea99072ba96e7ab`
* **Subnet ID:** `subnet-0d5246aaea2a63874` (`us-east-1a`)
* **Security Group Name:** `riverside-goods-sg`
* **Security Group ID:** `sg-0bf2c538893a7e3f8`
* **Instance ID:** `i-0fffaa16d7d06d3f2`
* **Instance Type:** `t3.micro`
* **AMI ID:** `ami-0b245cc5f82576748` (Amazon Linux 2023)
* **Initial Public IPv4:** `3.91.133.69`
* **Post-Lifecycle Public IPv4:** `52.91.114.64`
* **IAM Instance Profile:** `LabInstanceProfile`

---

## AWS Documentation Evidence

1. **AWS Security Group Ingress Behavior:** Security groups act as stateful firewalls at the hypervisor level. By default, newly created security groups allow all outbound traffic but deny all inbound traffic unless explicit ingress rules are configured.
2. **EC2 Public IPv4 Persistence:** Auto-assigned public IPv4 addresses are tied directly to the instance's network interface lifecycle for its current execution period. When an Amazon EC2 instance backed by Elastic Block Store (EBS) is stopped, the public IPv4 address is released back to Amazon's public IP pool and a new public IPv4 address is assigned upon restarting.
3. **Instance Metadata Service Version 2 (IMDSv2):** IMDSv2 provides session-oriented authentication to access instance metadata locally at `http://169.254.169.254`. It requires an initial HTTP `PUT` request with a specified TTL header to generate a token before metadata endpoints (such as `/latest/meta-data/instance-id`) can be queried.

---

## CloudShell Command Record

### 1. Identity & Environment Setup
```bash
aws sts get-caller-identity
export AWS_DEFAULT_REGION=$(aws configure get region)
if [ -z "$AWS_DEFAULT_REGION" ]; then
  export AWS_DEFAULT_REGION="us-east-1"
fi
echo "Operating in Region: $AWS_DEFAULT_REGION"

{
    "UserId": "AROA47DMA3EUH5P2UFBEB:user4742512=Jarvis_D._Anderson",
    "Account": "891432261928",
    "Arn": "arn:aws:sts::891432261928:assumed-role/voclabs/user4742512=Jarvis_D._Anderson"
}
Operating in Region: us-east-1

### 2.Network Infrastructure Discovery & Availability Zone Correction
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
echo "VPC ID: $VPC_ID"

SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=availability-zone,Values=us-east-1a" --query "Subnets[0].SubnetId" --output text)
echo "Subnet ID: $SUBNET_ID"

VPC ID: vpc-06ea99072ba96e7ab
Subnet ID: subnet-0d5246aaea2a63874

### 3. Security Group Creation & Identifier Recovery
SG_ID=$(aws ec2 create-security-group --group-name "riverside-goods-sg" --description "Lab SG with initially missing HTTP port 80" --vpc-id $VPC_ID --query 'GroupId' --output text)

# Handling duplicate security group entry recovery
SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=riverside-goods-sg" "Name=vpc-id,Values=$VPC_ID" --query "SecurityGroups[0].GroupId" --output text)
echo "Security Group ID: $SG_ID"

Security Group ID: sg-0bf2c538893a7e3f8

### 4. User Data Script Preparation & Instance Launch
cat << 'EOF' > userdata.txt
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Welcome to Riverside Goods Web Server</h1>" > /var/www/html/index.html
EOF

AMI_ID=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query "Parameters[0].Value" --output text)
echo "AMI ID: $AMI_ID"

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $SUBNET_ID \
  --security-group-ids $SG_ID \
  --user-data file://userdata.txt \
  --iam-instance-profile Name=LabInstanceProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Riverside-Web-Server}]' \
  --query 'Instances[0].InstanceId' --output text)

echo "Launched Instance ID: $INSTANCE_ID"
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "Public IPv4 Address: $PUBLIC_IP"

AMI ID: ami-0b245cc5f82576748
Launched Instance ID: i-0fffaa16d7d06d3f2
Public IPv4 Address: 3.91.133.69

### Baseline Evidence
##













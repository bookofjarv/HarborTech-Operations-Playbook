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

```

## Baseline Evidence
### 1. Instance Status Check

```bash

aws ec2 describe-instance-status --instance-ids $INSTANCE_ID

```

### Output

```bash
{
    "InstanceStatuses": [
        {
            "AvailabilityZone": "us-east-1a",
            "AvailabilityZoneId": "use1-az2",
            "InstanceId": "i-0fffaa16d7d06d3f2",
            "InstanceState": {
                "Code": 16,
                "Name": "running"
            },
            "InstanceStatus": {
                "Details": [{"Name": "reachability", "Status": "passed"}],
                "Status": "ok"
            },
            "SystemStatus": {
                "Details": [{"Name": "reachability", "Status": "passed"}],
                "Status": "ok"
            },
            "AttachedEbsStatus": {
                "Details": [{"Name": "reachability", "Status": "passed"}],
                "Status": "ok"
            }
        }
    ]
}

```

## 2. Pre-Fix Security Group Configuration

```bash

aws ec2 describe-security-groups --group-ids $SG_ID

Output

{
    "SecurityGroups": [
        {
            "GroupId": "sg-0bf2c538893a7e3f8",
            "GroupName": "riverside-goods-sg",
            "Description": "Lab SG with initially missing HTTP port 80",
            "VpcId": "vpc-06ea99072ba96e7ab",
            "IpPermissions": [],
            "IpPermissionsEgress": [
                {
                    "IpProtocol": "-1",
                    "IpRanges": [{"CidrIp": "0.0.0.0/0"}]
                }
            ]
        }
    ]
}
```
### 3. Baseline HTTP Reachability Failure
```bash
curl -m 5 -v http://$PUBLIC_IP

Output:
*   Trying 3.91.133.69:80...
* Connection timed out after 5001 milliseconds
* closing connection #0
curl: (28) Connection timed out after 5001 milliseconds


```
---

### Root-Cause Analysis

The instance status checks returned 2/2 ok (SystemStatus: ok, InstanceStatus: ok), confirming that the virtual machine was powered on, host physical hardware was operational, and basic network reachability at the hypervisor level was functional.

However, the external HTTP connection test to http://3.91.133.69:80 resulted in a connection timeout (curl: (28) Connection timed out). Cross-referencing this with the AWS control plane configuration showed that sg-0bf2c538893a7e3f8 contained an empty IpPermissions list, meaning zero inbound rules were configured.

Because AWS security groups drop unauthorized inbound packets silently at the hypervisor perimeter, network traffic attempting to reach TCP port 80 was dropped prior to reaching the instance operating system or the Apache web daemon. The issue was entirely isolated to a missing control-plane ingress rule on the security group.

---

### Corrective Action
To resolve the issue, an ingress rule allowing TCP port 80 from any source (0.0.0.0/0) was applied directly to the existing security group (sg-0bf2c538893a7e3f8). Rebuilding or redeploying the EC2 instance was not required.

```bash

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

Output:

{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-06cf7da689fc5526c",
            "GroupId": "sg-0bf2c538893a7e3f8",
            "GroupOwnerId": "891432261928",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}

```
---

## Verification Evidence

### 1. Control-Plane Ingress Rule Verification
```bash

aws ec2 describe-security-groups --group-ids $SG_ID --query "SecurityGroups[0].IpPermissions"

Output:

[
    {
        "IpProtocol": "tcp",
        "FromPort": 80,
        "ToPort": 80,
        "UserIdGroupPairs": [],
        "IpRanges": [
            {
                "CidrIp": "0.0.0.0/0"
            }
        ],
        "Ipv6Ranges": [],
        "PrefixListIds": []
    }
]

```

## 2. External HTTP Reachability Test

```bash
curl -m 5 http://$PUBLIC_IP

Output:
<h1>Welcome to Riverside Goods Web Server</h1>

```

## IMDSv2 and Guest Evidence
### AWS Systems Manager Session Manager was used to establish an interactive terminal session inside the running instance to verify internal guest OS health.

```bash

aws ssm start-session --target $INSTANCE_ID

Command execution inside SSM guest session:

# Verify Apache status locally
systemctl status httpd

# Verify local web server HTTP response
curl http://localhost

# Obtain IMDSv2 session token and query Instance ID metadata
TOKEN=$(curl -s -X PUT "[http://169.254.169.254/latest/api/token](http://169.254.169.254/latest/api/token)" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
IMDS_INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" [http://169.254.169.254/latest/meta-data/instance-id](http://169.254.169.254/latest/meta-data/instance-id))
echo "Local IMDSv2 Instance ID: $IMDS_INSTANCE_ID"

exit

Session Output Transcript:
Starting session with SessionId: user4742512=Jarvis_D._Anderson-4n8zdgt8i4dfrci8atfcy25xpe
sh-5.2$ systemctl status httpd
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Wed 2026-09-30 03:12:58 UTC; 4min 46s ago
   Main PID: 3582 (httpd)

sh-5.2$ curl http://localhost
<h1>Welcome to Riverside Goods Web Server</h1>

sh-5.2$ TOKEN=$(curl -s -X PUT "[http://169.254.169.254/latest/api/token](http://169.254.169.254/latest/api/token)" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
sh-5.2$ IMDS_INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" [http://169.254.169.254/latest/meta-data/instance-id](http://169.254.169.254/latest/meta-data/instance-id))
sh-5.2$ echo "Local IMDSv2 Instance ID: $IMDS_INSTANCE_ID"
Local IMDSv2 Instance ID: i-0fffaa16d7d06d3f2
sh-5.2$ exit

```
---

### Significance of Internal Evidence:

External AWS CLI outputs confirm control-plane status and network filtering configurations. Internal guest evidence collected via SSM and IMDSv2 confirms that user-data execution succeeded, local web services are running, and internal hypervisor communication functions properly.

---
### Stop/Start Lifecycle Test
A stop and start lifecycle test was performed to verify data persistence on the EBS root volume and observe public IP behavior.

```bash

# Record initial IP
echo "Pre-stop IP: $PUBLIC_IP"

# Stop instance and wait for state transition
aws ec2 stop-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID

# Restart instance and wait for state transition
aws ec2 start-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# Retrieve updated public IPv4 address
NEW_PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "Post-start New Public IPv4: $NEW_PUBLIC_IP"

# Retest web application reachability on the new IP address
curl -m 5 http://$NEW_PUBLIC_IP

Output:
Pre-stop IP: 3.91.133.69
Post-start New Public IPv4: 52.91.114.64
<h1>Welcome to Riverside Goods Web Server</h1>

```
---
### Lifecycle Analysis
What Persisted: The underlying EBS root volume retained all system configurations, installed binaries (httpd), and file data (/var/www/html/index.html). The Instance ID (i-0fffaa16d7d06d3f2) and security group mapping remained unchanged.

What Changed: The initial public IPv4 address (3.91.133.69) was released upon stopping and replaced with a new public IPv4 address (52.91.114.64) upon restart. This behavior confirms that standard auto-assigned public IP addresses are ephemeral across stop/start operations.

---

### Cleanup Evidence
The instance was terminated and the lab security group was removed after verifying instance termination.

```bash

aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID

aws ec2 delete-security-group --group-id $SG_ID
echo "Lab cleanup completed."

Output:

{
    "TerminatingInstances": [
        {
            "InstanceId": "i-0fffaa16d7d06d3f2",
            "CurrentState": {
                "Code": 32,
                "Name": "shutting-down"
            },
            "PreviousState": {
                "Code": 16,
                "Name": "running"
            }
        }
    ]
}
{
    "Return": true,
    "GroupId": "sg-0bf2c538893a7e3f8"
}
Lab cleanup completed.
```

---

## Escalation and Change-Control Notes
Escalation Path: Resolved at Tier 2. No Tier 3 escalation was necessary as the issue did not stem from AMI corruption, hardware failure, or VPC route table misconfigurations.

Justification Against Rebuilding: Rebuilding or re-provisioning an EC2 instance introduces unnecessary service disruption, loses system state, and fails to address control-plane configuration issues. Targeted security group ingress updates resolve the issue immediately without instance downtime or resource re-creation.

---

## Lessons Learned
Distinguish Network Filtering from Host Health: Passing AWS 2/2 status checks indicates healthy physical infrastructure and hypervisor responsiveness, but does not guarantee that network firewalls allow application traffic.

Account for Public IP Ephemerality: Auto-assigned public IPv4 addresses change when EBS-backed instances are stopped and started. Production workloads requiring persistent external access should use Elastic IP addresses (EIP) or DNS names managed by Route 53.

Isolate Verification Scopes: Verification must occur at both the control plane (verifying security group rules) and within the guest operating system (checking daemon status via SSM Session Manager).

---

## Professional Vocabulary
Control Plane: The AWS administrative management layer used to manage and configure cloud infrastructure resources (e.g., Security Groups, VPCs).

Data Plane / Workload Path: The actual network path over which end-user traffic flows to interact with hosted services.

Security Group Ingress Rule: A stateful control-plane firewall rule that defines allowed inbound traffic parameters (protocol, port range, source IP).

Instance Metadata Service Version 2 (IMDSv2): A secure, session-oriented local endpoint (169.254.169.254) used by EC2 instances to retrieve runtime data.

AWS Systems Manager (SSM) Session Manager: A fully managed capability providing secure, IAM-authorized shell management of EC2 instances without opening inbound SSH ports.

EBS Persistence: Non-volatile block storage that persists data independently of the EC2 instance lifecycle, preserving file systems across instance reboots and stop/start cycles.





































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

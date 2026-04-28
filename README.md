# Highly Available Multi-AZ VPC Architecture Lab

> AWS CloudFormation · EC2 · Systems Manager · NAT Gateways · Apache HTTP Server

---

## Overview

This project deploys a **production-grade, fault-tolerant VPC** on AWS using Infrastructure as Code (CloudFormation). The architecture spans **two Availability Zones** with redundant public and private subnets, dual NAT Gateways, and four EC2 instances across a web tier and application tier — all managed exclusively via **AWS Systems Manager Session Manager** (no SSH keys required).

---

## Architecture Diagram

```
                          ┌─────────────────────────────────────────────────────┐
                          │                    AWS Region                        │
                          │                                                       │
                          │  ┌──────────────────────────────────────────────┐   │
                          │  │             VPC  10.0.0.0/16                 │   │
                          │  │                                               │   │
                          │  │        Internet Gateway (IGW)                │   │
                          │  │                   │                           │   │
                          │  │    ┌──────────────┴──────────────┐           │   │
                          │  │    │                             │            │   │
                          │  │  AZ 1                         AZ 2           │   │
                          │  │                                               │   │
                          │  │  ┌──────────────┐  ┌──────────────┐         │   │
                          │  │  │Public Subnet1│  │Public Subnet2│         │   │
                          │  │  │ 10.0.1.0/24  │  │ 10.0.2.0/24  │         │   │
                          │  │  │              │  │              │         │   │
                          │  │  │ [Web Srv 1]  │  │ [Web Srv 2]  │         │   │
                          │  │  │  Apache      │  │  Apache      │         │   │
                          │  │  │              │  │              │         │   │
                          │  │  │  NAT GW 1    │  │  NAT GW 2    │         │   │
                          │  │  └──────┬───────┘  └──────┬───────┘         │   │
                          │  │         │                  │                 │   │
                          │  │  ┌──────▼───────┐  ┌──────▼───────┐         │   │
                          │  │  │Private Subnet│  │Private Subnet│         │   │
                          │  │  │ 10.0.11.0/24 │  │ 10.0.12.0/24 │         │   │
                          │  │  │              │  │              │         │   │
                          │  │  │ [App Srv 1]  │  │ [App Srv 2]  │         │   │
                          │  │  │  (private)   │  │  (private)   │         │   │
                          │  │  └──────────────┘  └──────────────┘         │   │
                          │  └──────────────────────────────────────────────┘   │
                          └─────────────────────────────────────────────────────┘
```

---

## Resources Deployed

| Resource | Count | Details |
|---|---|---|
| VPC | 1 | 10.0.0.0/16, DNS enabled |
| Internet Gateway | 1 | Attached to VPC |
| Public Subnets | 2 | One per AZ (10.0.1.0/24, 10.0.2.0/24) |
| Private Subnets | 2 | One per AZ (10.0.11.0/24, 10.0.12.0/24) |
| NAT Gateways | 2 | One per AZ (no cross-AZ dependency) |
| Elastic IPs | 2 | One per NAT Gateway |
| Route Tables | 3 | 1 public + 2 private (AZ-local) |
| Security Groups | 2 | Web Tier SG + App Tier SG |
| IAM Role | 1 | AmazonSSMManagedInstanceCore |
| EC2 Instances | 4 | 2 web (public) + 2 app (private) |

---

## Security Design

### Web Tier Security Group
| Rule | Direction | Protocol | Port | Source |
|---|---|---|---|---|
| HTTP | Inbound | TCP | 80 | 0.0.0.0/0 |
| ICMP Ping | Inbound | ICMP | -1 | 10.0.0.0/16 |
| All traffic | Outbound | All | All | 0.0.0.0/0 |

### App Tier Security Group
| Rule | Direction | Protocol | Port | Source |
|---|---|---|---|---|
| ICMP Ping | Inbound | ICMP | -1 | 10.0.0.0/16 |
| All traffic | Outbound | All | All | 0.0.0.0/0 |

> ⚠️ **No SSH (port 22) is open on any instance.** All access is exclusively through AWS Systems Manager Session Manager.

---

## Prerequisites

- AWS CLI installed and configured (`aws configure`)
- IAM permissions to create VPC, EC2, IAM, and CloudFormation resources
- AWS region with at least 2 Availability Zones

---

## Deployment

### Step 1 — Clone this repository

```bash
git clone https://github.com/peprah12git/VPC-using-cfn-LAB.git
cd VPC-using-cfn-LAB
```

### Step 2 — Deploy the stack

```bash
aws cloudformation deploy \
  --template-file cloudformation-template.yml \
  --stack-name ha-vpc-lab \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### Step 3 — Retrieve outputs

```bash
aws cloudformation describe-stacks \
  --stack-name ha-vpc-lab \
  --query "Stacks[0].Outputs" \
  --output table
```

---

## Connecting to Instances (Session Manager)

All EC2 instances are managed **exclusively via AWS Systems Manager Session Manager**. No SSH keys are required or allowed.

### Option A — AWS Management Console

1. Open the [AWS Systems Manager Console](https://console.aws.amazon.com/systems-manager/session-manager/sessions)
2. Click **Start session**
3. Select the instance (e.g., `HA-VPC-Lab-Web-Server-1-AZ1`)
4. Click **Start session**

### Option B — AWS CLI

```bash
# Get instance IDs from stack outputs
aws cloudformation describe-stacks \
  --stack-name ha-vpc-lab \
  --query "Stacks[0].Outputs[?contains(OutputKey,'InstanceId')].{Key:OutputKey,Value:OutputValue}" \
  --output table

# Start a session with an instance
aws ssm start-session --target i-0123456789abcdef0
```

---

## Validation

### 1. Test Web Servers (HTTP)

Get the public IPs from stack outputs, then open in a browser:

```
http://<WebServer1PublicIP>
http://<WebServer2PublicIP>
```

Both should display the HTML page with your full name and lab name.

### 2. Verify Apache is Running (via Session Manager)

```bash
# Connect to a web server via Session Manager, then:
systemctl status httpd
curl localhost
```

### 3. Ping Between Instances (ICMP Validation)

```bash
# From a web server, ping the private app servers:
ping 10.0.11.x   # App Server 1
ping 10.0.12.x   # App Server 2
```

### 4. Verify Private Instances Have Internet Access via NAT

```bash
# Connect to a private app server via Session Manager, then:
curl -s https://checkip.amazonaws.com
# Should return the NAT Gateway's Elastic IP — NOT the instance's private IP
```

### 5. Verify No Cross-AZ NAT Dependency

- Private Subnet 1 (AZ1) routes `0.0.0.0/0` → NAT Gateway 1 (in AZ1)
- Private Subnet 2 (AZ2) routes `0.0.0.0/0` → NAT Gateway 2 (in AZ2)

```bash
# Check route tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<your-vpc-id>" \
  --query "RouteTables[*].{RTName:Tags[?Key=='Name']|[0].Value,Routes:Routes}" \
  --output json
```

---

## Parameters

You can override default values at deploy time:

```bash
aws cloudformation deploy \
  --template-file cloudformation-template.yml \
  --stack-name ha-vpc-lab \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
      EnvironmentName=MyLab \
      InstanceType=t3.small \
      VpcCIDR=10.1.0.0/16
```

| Parameter | Default | Description |
|---|---|---|
| `EnvironmentName` | `HA-VPC-Lab` | Prefix for all resource names |
| `VpcCIDR` | `10.0.0.0/16` | VPC CIDR block |
| `PublicSubnet1CIDR` | `10.0.1.0/24` | Public Subnet AZ1 |
| `PublicSubnet2CIDR` | `10.0.2.0/24` | Public Subnet AZ2 |
| `PrivateSubnet1CIDR` | `10.0.11.0/24` | Private Subnet AZ1 |
| `PrivateSubnet2CIDR` | `10.0.12.0/24` | Private Subnet AZ2 |
| `InstanceType` | `t3.micro` | EC2 instance type |
| `AmazonLinuxAMI` | `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64` | Amazon Linux 2023 AMI (resolved dynamically via SSM) |

---

## Teardown

```bash
aws cloudformation delete-stack --stack-name ha-vpc-lab
```

> **Note:** Wait for the stack deletion to complete before verifying cleanup. NAT Gateways and EIPs are automatically removed.

---

## Key Design Decisions

1. **No SSH anywhere** — Zero key pair references in the template. Session Manager is the only access path.
2. **AZ-local NAT Gateways** — Each private subnet uses the NAT Gateway in its own AZ. Eliminates cross-AZ data transfer charges and single points of failure.
3. **Dynamic AMI resolution** — Uses an SSM Parameter Store path (`/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`) so the template always uses the latest Amazon Linux 2023 AMI without manual updates.
4. **Least privilege SGs** — Web tier only exposes port 80 and ICMP. App tier exposes only ICMP within the VPC.
5. **CAPABILITY_NAMED_IAM** — Required because the template creates a named IAM role and instance profile for SSM.

---

## Author

**Emmanuel Mensah Peprah**  
Lab: Highly Available Multi-AZ VPC Architecture Lab
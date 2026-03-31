# Secure Private Infrastructure Deployment
## Bastion Access and IAM Role-Based Architecture
**Region:** ap-west-2 | **Internship Project — AWS Cloud Security**

---

## 1. Project Overview

This project implements a production-grade secure AWS infrastructure for a fintech company. Application servers run in private subnets with no public IP, accessible only through a hardened Bastion Host. All EC2 instances use IAM Roles — no hardcoded credentials anywhere.

---

## 2. Architecture

![Architecture Diagram](/images/architecture.png)

### Network Layout

| Component | Type | CIDR / Detail |
|---|---|---|
| fintech-vpc | VPC | 10.0.0.0/16 |
| public-subnet-1 | Public Subnet | 10.0.1.0/24 — us-west-2a |
| public-subnet-2 | Public Subnet | 10.0.2.0/24 — us-west-2b |
| private-subnet-1 | Private Subnet | 10.0.3.0/24 — us-west-2a |
| private-subnet-2 | Private Subnet | 10.0.4.0/24 — us-west-2b |
| fintech-igw | Internet Gateway | Attached to fintech-vpc |
| fintech-nat-gw | NAT Gateway | In public-subnet-1, Elastic IP attached |

---

## 3. Security Design Decisions

### Bastion Host
- Deployed in public subnet with a static public IP
- SSH (port 22) restricted to a single trusted IP only (`your-ip/32`)
- Password-based login disabled — key-pair authentication only
- Security group has no other inbound rules

### Private Application Server
- No public IP — completely unreachable from the internet
- SSH allowed only from `sg-bastion` (security group reference, not IP)
- Outbound traffic routes through NAT Gateway only

### Security Groups

| Security Group | Rule | Source |
|---|---|---|
| sg-bastion | SSH — Port 22 | Your IP (x.x.x.x/32) |
| sg-appserver | SSH — Port 22 | sg-bastion |
| sg-appserver | HTTP — Port 80 | sg-bastion |

### IAM Role (No Hardcoded Credentials)
- Role `ec2-s3-readonly-role` created with `AmazonS3ReadOnlyAccess` policy
- Role attached to private EC2 via instance profile
- No `~/.aws/credentials` file on server — temporary credentials via instance metadata

---

## 4. Setup Steps

| Step | Action | Location |
|---|---|---|
| 1 | Create VPC (10.0.0.0/16) | VPC Console |
| 2 | Create 4 subnets (2 public, 2 private) | VPC → Subnets |
| 3 | Create and attach Internet Gateway | VPC → Internet Gateways |
| 4 | Create NAT Gateway with Elastic IP in public subnet | VPC → NAT Gateways |
| 5 | Create public route table — 0.0.0.0/0 → IGW | VPC → Route Tables |
| 6 | Create private route table — 0.0.0.0/0 → NAT GW | VPC → Route Tables |
| 7 | Create sg-bastion (SSH from your IP only) | EC2 → Security Groups |
| 8 | Create sg-appserver (SSH + HTTP from sg-bastion) | EC2 → Security Groups |
| 9 | Launch Bastion Host in public-subnet-1 | EC2 → Launch Instance |
| 10 | Launch App Server in private-subnet-1 (no public IP) | EC2 → Launch Instance |
| 11 | Create IAM Role with S3ReadOnly, attach to App Server | IAM → Roles |

---

## 5. Access Flow

```bash
# On local machine
chmod 400 fintech-key.pem
scp -i fintech-key.pem fintech-key.pem ec2-user@<BASTION_PUBLIC_IP>:~/.ssh/
ssh -i fintech-key.pem ec2-user@<BASTION_PUBLIC_IP>

# Inside bastion — hop to private app server
chmod 400 ~/.ssh/fintech-key.pem
ssh -i ~/.ssh/fintech-key.pem ec2-user@<APP_SERVER_PRIVATE_IP>
```

Access flow: `Local Machine → Bastion Host (public) → App Server (private)`

---

## 6. Screenshots Index

| # | File Name | What It Shows |
|---|---|---|
| 1 | architecture.png | Full AWS architecture diagram |
| 2 | 01_vpc_details.png | VPC page — CIDR 10.0.0.0/16 |
| 3 | 02_subnets_list.png | All 4 subnets with names and CIDRs |
| 4 | 03_igw_attached.png | Internet Gateway in Attached state |
| 5 | 04_nat_gateway_available.png | NAT Gateway Available + Elastic IP |
| 6 | 05_public_route_table.png | Route: 0.0.0.0/0 → IGW |
| 7 | 06_private_route_table.png | Route: 0.0.0.0/0 → NAT GW |
| 8 | 07_sg_bastion_inbound.png | sg-bastion: SSH from your IP only |
| 9 | 08_sg_appserver_inbound.png | sg-appserver: SSH + HTTP from sg-bastion |
| 10 | 09_bastion_ec2_details.png | Bastion: public IP + public subnet |
| 11 | 10_appserver_ec2_details.png | App server: no public IP + private subnet |
| 12 | 11_ssh_into_bastion.png | Terminal: successful SSH to bastion |
| 13 | 12_ssh_bastion_to_appserver.png | Terminal: hop from bastion to private EC2 |
| 14 | 13_direct_ssh_timeout.png | Terminal: direct SSH to private IP times out |
| 15 | 14_iam_role_policy.png | IAM role with AmazonS3ReadOnlyAccess attached |
| 16 | 15_iam_role_attached_ec2.png | App server Security tab showing role attached |

---

## 7. Lessons Learned

- Never assign public IPs to application servers — use private subnets and a Bastion
- Security group chaining is more secure than IP-based rules for internal traffic
- IAM Instance Profiles eliminate access keys entirely — nothing to leak or rotate manually
- NAT Gateway gives private instances outbound internet without any inbound exposure
- Route tables are the key control point — private subnets must never route directly to IGW
- Always restrict Bastion SSH to a specific `/32` IP, never `0.0.0.0/0`

---

## 8. Technologies Used

| Technology | Purpose |
|---|---|
| Amazon VPC | Network isolation — custom CIDR, subnets, route tables |
| Amazon EC2 | Bastion Host and private application server |
| Internet Gateway | Public internet access for public subnet resources |
| NAT Gateway | Outbound-only internet for private subnet instances |
| IAM Roles | Credential-free AWS service access from EC2 |
| Amazon S3 | Storage accessed via IAM role — no access keys |
| Security Groups | Stateful firewall — enforces least privilege access |
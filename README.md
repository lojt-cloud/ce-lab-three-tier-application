
#  AWS 3-Tier Enterprise Cloud Architecture

A multi-AZ, highly available, and secure 3-tier enterprise cloud infrastructure deployed on AWS using the AWS CLI. This project demonstrates strict network isolation, defense-in-depth security group chaining, and automated traffic distribution across Availability Zones.

---

##  Architecture Overview

The system is deployed within a custom VPC (`10.0.0.0/16`) spanned across two Availability Zones (`us-east-1a` and `us-east-1b`) for fault tolerance and high availability:

```text
[ Internet Clients ]
        │
        ▼ (Port 80 Ingress)
┌────────────────────────────────────────────────────────────────────────┐
│ TIER 1: PRESENTATION LAYER (Public Subnets: 10.0.1.0/24 & 10.0.2.0/24) │
│                        Application Load Balancer                       │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ (Port 80 Internal Route)
┌───────────────────────────────────▼────────────────────────────────────┐
│ TIER 2: APPLICATION LAYER (Private Subnets: 10.0.11.0/24 & 10.0.12.0/24)│
│  [ App Server 1a (t3.micro) ]        [ App Server 1b (t3.micro) ]     │
│                 │ (Outbound updates via NAT Gateway)                   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ (Port 3306 Internal MySQL)
┌───────────────────────────────────▼────────────────────────────────────┐
│ TIER 3: DATA LAYER (Isolated Subnets: 10.0.21.0/24 & 10.0.22.0/24)     │
│                     [ Database Host (10.0.21.10) ]                      │
└────────────────────────────────────────────────────────────────────────┘


 Security & Networking Highlights

Tier Isolation: Private application subnets have no public IP addresses. Tier 3 database subnets have zero internet route paths (no IGW, no NAT Gateway).

Security Group Chaining: Rules do not rely on open CIDRs; ingress permissions are chained directly between Security Groups (alb-sg ➔ app-tier-sg ➔ db-tier-sg).

High Availability & Health Probing: The Application Load Balancer distributes HTTP traffic round-robin across instances in two AZs and actively probes /health every 15 seconds.

IMDSv2 Enforcement: EC2 boot scripts use AWS Instance Metadata Service Version 2 session tokens for secure token-based metadata access.

 Repository Structure
.
├── README.md
├── app
│   ├── db-sim.sh
│   ├── server.js
│   └── userdata.sh
├── app-userdata.sh
├── architecture
│   ├── ARCHITECTURE.md
│   ├── NETWORK-DESIGN.md
│   ├── SECURITY.md
│   └── architecture-diagram.png
├── config
│   ├── instances.txt
│   ├── load-balancer.txt
│   ├── security-groups.txt
│   └── vpc-subnets.txt
├── lab-instructions.md
├── screenshots
└── tests
    ├── end-to-end-test.md
    ├── security-test.md
    └── traffic-flow-test.md


Verification & End-to-End Testing

1. Target Health Check:
aws elbv2 describe-target-health --target-group-arn <TARGET_GROUP_ARN>
# Returns: State: "healthy" across both targets

2. Load Balancing Test: 
curl -s "http://<ALB_DNS_NAME>/api/stats"
# Demonstrates alternating responses between app-server-1a and app-server-1b

Project Insights & AI Attribution
Note on Architecture Focus & Automation:
My focus as a Cloud Engineer is centered around system design, cloud infrastructure, network topology, security boundaries, and high availability.

Because shell scripting and application coding are not my core specialization, I leveraged Artificial Intelligence (AI) to assist in generating bash automation scripts, user-data bootstrap configurations, and diagnostic tools. This allowed me to focus my efforts on configuring the AWS cloud environment, validating security group chaining, verifying route tables, and troubleshooting enterprise infrastructure patterns.
# AWS Highly Available Web Application

A production-grade, multi-AZ web application infrastructure built on AWS demonstrating high availability, network security, and automatic scalability — core skills for cloud engineering and cybersecurity roles.

---

## Architecture Overview

```
                          ┌─────────────────────────────────────────────────┐
                          │                  VPC 10.0.0.0/16                │
                          │                                                 │
          Internet ──── IGW                                                 │
                          │                                                 │
                    ┌─────┴──────────────────────────────┐                  │
                    │         Public Subnets              │                 │
                    │  ┌──────────────┐ ┌─────────────┐  │                  │
                    │  │  ALB Node    │ │  ALB Node   │  │                  │
                    │  │  AZ-1a       │ │  AZ-1b      │  │                  │
                    │  └──────────────┘ └─────────────┘  │                  │
                    │  ┌──────────────┐ ┌─────────────┐  │                  │
                    │  │ NAT Gateway  │ │ NAT Gateway │  │                  │
                    │  │  AZ-1a       │ │  AZ-1b      │  │                  │
                    │  └──────────────┘ └─────────────┘  │                  │
                    └─────┬──────────────────────────────┘                  │
                          │ ALB routes traffic                              │
                    ┌─────┴──────────────────────────────┐                  │
                    │         Private Subnets             │                 │
                    │  ┌──────────────┐ ┌─────────────┐  │                  │
                    │  │  EC2 Web     │ │  EC2 Web    │  │                  │
                    │  │  Server      │ │  Server     │  │                  │
                    │  │  AZ-1a       │ │  AZ-1b      │  │                  │
                    │  └──────────────┘ └─────────────┘  │                  │
                    │       Auto Scaling Group            │                 │
                    │       Min:1  Desired:2  Max:4       │                 │
                    └────────────────────────────────────┘                  │
                          │                                                 │
                          └─────────────────────────────────────────────────┘
```

> See [`architecture/`](./architecture/) for full diagram screenshots.

---

## What This Demonstrates

| Skill | Implementation |
|---|---|
| Cloud networking | Custom VPC, CIDR planning, public/private subnet segmentation |
| Security design | Security groups with least-privilege, no direct EC2 internet exposure |
| High availability | Multi-AZ deployment, ALB health checks, automatic failover |
| Scalability | Auto Scaling Group with target tracking (CPU utilization) |
| Network routing | Internet Gateway, NAT Gateway, custom route tables |
| Infrastructure design | Separation of concerns between tiers |

---

## AWS Services Used

- **VPC** — Custom virtual network with CIDR `10.0.0.0/16`
- **Subnets** — 2 public (`10.0.1.0/24`, `10.0.2.0/24`) and 2 private (`10.0.3.0/24`, `10.0.4.0/24`) across 2 AZs
- **Internet Gateway** — Enables inbound/outbound internet for public subnets
- **NAT Gateway** — Enables outbound-only internet access for private EC2 instances
- **Application Load Balancer** — Internet-facing, distributes traffic across EC2 instances
- **EC2** — Web servers running Apache httpd, launched via Launch Template
- **Auto Scaling Group** — Maintains instance count, replaces failed instances, scales on CPU
- **Security Groups** — ALB SG (open to internet) and EC2 SG (open to ALB only)
- **CloudWatch** — Monitors CPU metrics to trigger scaling events

---

## Network Design

### Subnet Layout

| Subnet | CIDR | AZ | Type | Hosts |
|---|---|---|---|---|
| public-subnet-1a | 10.0.1.0/24 | us-east-1a | Public | ALB, NAT GW |
| public-subnet-1b | 10.0.2.0/24 | us-east-1b | Public | ALB, NAT GW |
| private-subnet-1a | 10.0.3.0/24 | us-east-1a | Private | EC2 instances |
| private-subnet-1b | 10.0.4.0/24 | us-east-1b | Private | EC2 instances |

### Route Tables

**Public route table** (associated with both public subnets):
| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | Internet Gateway |

**Private route table** (associated with both private subnets):
| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | NAT Gateway |

### Security Groups

**alb-sg** (Application Load Balancer):
| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | 0.0.0.0/0 |
| Inbound | HTTPS | 443 | 0.0.0.0/0 |
| Outbound | All | All | 0.0.0.0/0 |

**ec2-sg** (EC2 instances):
| Direction | Protocol | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | alb-sg (SG reference) |
| Outbound | All | All | 0.0.0.0/0 |

> EC2 instances accept traffic **only** from the ALB security group — not from the internet directly.

---

## EC2 Bootstrap Script

Each EC2 instance runs this user data script on first boot, installing Apache and serving a page that identifies the host:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
```

See the full script: [`scripts/user_data.sh`](./scripts/user_data.sh)

---

## Auto Scaling Configuration

| Setting | Value |
|---|---|
| Minimum capacity | 1 |
| Desired capacity | 2 |
| Maximum capacity | 4 |
| Scaling policy | Target tracking |
| Metric | Average CPU utilization |
| Target value | 50% |
| Health check type | ELB |

When CPU utilization across the group exceeds 50%, CloudWatch triggers a scale-out event and the ASG launches new instances from the launch template. Instances are placed in whichever private subnet has the least load, spreading across AZs automatically.

---

## Traffic Flow

**Inbound (user request):**
```
User → Internet → Internet Gateway → ALB (public subnet) → EC2 (private subnet)
```

**Outbound (EC2 update/install):**
```
EC2 (private subnet) → NAT Gateway (public subnet) → Internet Gateway → Internet
```

**Health check:**
```
ALB → HTTP GET / → EC2 every 30s → unhealthy threshold 2 → deregister + ASG replaces
```

---

## High Availability Design

This architecture is designed to survive an Availability Zone failure:

1. Resources exist in both `us-east-1a` and `us-east-1b`
2. The ALB automatically stops routing to instances in a failed AZ
3. The Auto Scaling Group detects unhealthy instances and launches replacements in the healthy AZ
4. NAT Gateways exist in both AZs so private instances in either AZ maintain outbound access
5. Desired capacity of 2 means both AZs are always active under normal conditions — no warm-up delay on failover

---

## Screenshots

See [`docs/screenshots/`](./docs/screenshots/) for:

- `01-vpc.png` — VPC created with correct CIDR
- `02-subnets.png` — All 4 subnets across 2 AZs
- `03-igw.png` — Internet Gateway attached to VPC
- `04-nat-gateway.png` — NAT Gateway in public subnet with Elastic IP
- `05-route-tables.png` — Public and private route tables with routes
- `06-security-groups.png` — ALB and EC2 security group rules
- `07-launch-template.png` — Launch template configuration
- `08-alb.png` — Load balancer in active state
- `09-target-group.png` — Target group with healthy targets
- `10-asg.png` — Auto Scaling Group with instance activity
- `11-ec2-instances.png` — Running instances in private subnets
- `12-working-app.png` — Browser showing app served through ALB DNS name

---

## Project Structure

```
aws-ha-webapp/
├── README.md                   # This file
├── architecture/
│   └── architecture-diagram.png
├── docs/
│   └── screenshots/            # AWS console screenshots (see list above)
├── scripts/
│   └── user_data.sh            # EC2 bootstrap script
└── terraform/
    └── main.tf                 # Infrastructure as Code (optional extension)
```

---

## Key Security Decisions

**Why are EC2 instances in private subnets?**
Direct internet access to backend servers is unnecessary and increases attack surface. All legitimate traffic comes through the ALB, which handles TLS termination and can be fronted with AWS WAF.

**Why reference the ALB security group instead of using a CIDR range?**
Using a security group reference means only resources actually attached to `alb-sg` can reach EC2 — not any IP that happens to fall in a given range. This is more secure and doesn't break if the ALB scales.

**Why Zonal NAT Gateways instead of Regional?**
Zonal NAT Gateways give explicit control over which AZ traffic exits from, keeping traffic within its AZ and reducing cross-AZ data transfer costs. Each private subnet routes to the NAT Gateway in its own AZ.

---

## Skills Demonstrated

`AWS` `VPC` `EC2` `ALB` `Auto Scaling` `NAT Gateway` `Security Groups` `Network Segmentation` `High Availability` `Fault Tolerance` `Cloud Security` `Infrastructure Design` `CloudWatch`

# AWS VPC + Auto Scaling Group + Application Load Balancer — Level Up Bank

## Business Use Case

Level Up Bank is migrating its entire network infrastructure from on-premises
to AWS to improve scalability, reliability, and security. After successfully
moving a single server to EC2, the bank now builds a full cloud network — a
custom VPC with auto-scaling web servers behind a load balancer.

**Why move the full network to AWS?**

- **Scalability** — Auto Scaling adjusts instance count automatically based on
  real-time demand, no hardware procurement required
- **Reliability** — Instances spread across multiple availability zones ensure
  the bank stays online if one zone goes down
- **Security** — VPC isolation, layered security groups, and traffic routing
  through a load balancer prevent direct exposure of web servers
- **Cost savings** — Pay only for instances running; scale down during off-peak
  hours instead of paying for idle on-premises hardware
- **Flexibility** — New instances launch automatically from a template, fully
  configured, with no manual setup

---

## Architecture

![AWS Architecture Diagram](docs/AWS-architecture%20(2).png)

---

## Tools & Services Used

- AWS VPC (Virtual Private Cloud)
- AWS Subnets (Public — 3 Availability Zones)
- AWS Internet Gateway
- AWS Route Tables
- AWS Security Groups
- AWS Application Load Balancer (ALB)
- AWS Target Group
- AWS Auto Scaling Group (ASG)
- AWS Launch Template
- Amazon EC2 (t2.micro)
- Amazon Linux 2023
- Apache HTTP Server (httpd)
- AWS Console (Console method)

---

## Prerequisites

- AWS Account with VPC, EC2, and ALB permissions
- AWS CLI installed and configured (for CLI method)
- Basic understanding of CIDR notation and subnetting

---

## Implementation — Console Method

### Step 1: Create the VPC

**Configuration:**

| Setting | Value | Reason |
|---|---|---|
| Resources to create | VPC only | Build each component manually for full understanding |
| Name | `levelup-bank-vpc` | Identifiable across all project resources |
| IPv4 CIDR | `10.10.0.0/16` | Private address space — 65,536 available IPs |
| IPv6 | None | Not required for this project |
| Tenancy | Default | Shared hardware — free tier eligible |

> **Why VPC only and not VPC & More:** The wizard hides the creation of subnets,
> route tables, and internet gateways. Building each manually teaches how they
> connect — knowledge required when troubleshooting real production environments.

> **Security Note:** A custom VPC is isolated by default. Nothing enters or
> leaves until you explicitly create and attach an Internet Gateway and configure
> routing. This is the correct starting state.

---

### Step 2: Create Three Public Subnets

One subnet per availability zone to enable high availability across the region.

| Name | CIDR Block | Availability Zone |
|---|---|---|
| `levelup-public-1a` | `10.10.1.0/24` | us-east-1a |
| `levelup-public-1b` | `10.10.2.0/24` | us-east-1b |
| `levelup-public-1c` | `10.10.3.0/24` | us-east-1c |

> **Two most important settings:**
> - **CIDR block must fall inside the VPC CIDR** — `10.10.1.0/24` is valid
>   inside `10.10.0.0/16`. If they don't match, AWS rejects the subnet
>   creation outright
> - **One subnet per AZ** — this is what enables high availability. If
>   `us-east-1a` goes down, the ASG has healthy instances in `1b` and `1c`
>   to keep the bank running

> **Note:** Subnets are not publicly accessible at this stage. A subnet is
> only truly "public" after attaching an Internet Gateway and configuring
> a Route Table to route traffic through it.

---

### Step 3: Create Internet Gateway and Route Table

**Internet Gateway:**

| Setting | Value |
|---|---|
| Name | `levelup-bank-igw` |
| Attached to VPC | `levelup-bank-vpc` |

> **Critical:** An Internet Gateway that is not attached to a VPC does
> nothing. AWS will let you create it without warning. Always confirm
> the state shows **Attached** before moving on.

**Route Table:**

| Setting | Value |
|---|---|
| Name | `levelup-public-rt` |
| VPC | `levelup-bank-vpc` |

**Routes configured:**

| Destination | Target | Purpose |
|---|---|---|
| `10.10.0.0/16` | local | Internal VPC traffic stays inside — no internet hop |
| `0.0.0.0/0` | `levelup-bank-igw` | All other traffic exits through the Internet Gateway |

**Subnet associations:** `levelup-public-1a`, `levelup-public-1b`, `levelup-public-1c`

> **Two most important lines:**
> - **`0.0.0.0/0` → IGW** — without this route, instances have no path to
>   the internet. ALB health checks fail, and the user data script cannot
>   download Apache during first boot
> - **`10.10.0.0/16` → local** — AWS adds this automatically. It keeps traffic
>   between instances inside the VPC, avoiding an unnecessary round trip to
>   the internet and back

---

### Step 4: Create Security Groups

Two security groups are required and must be created in order — the web server
group references the load balancer group as its traffic source.

**Security Group 1 — Load Balancer (create first):**

| Setting | Value |
|---|---|
| Name | `levelup-alb-sg` |
| Description | Allow HTTP from internet to ALB |
| VPC | `levelup-bank-vpc` |

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 80 | TCP | `0.0.0.0/0` | Accept HTTP traffic from the public internet |

**Security Group 2 — Web Server (create second):**

| Setting | Value |
|---|---|
| Name | `levelup-webserver-sg` |
| Description | Allow HTTP only from ALB |
| VPC | `levelup-bank-vpc` |

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 80 | TCP | `levelup-alb-sg` | Accept traffic only if it came through the ALB |

> **Security Note:** The web server security group uses another security group
> as its source — not an IP range. This means AWS dynamically tracks every
> resource attached to `levelup-alb-sg`. If the ALB's IPs change (which they
> do — ALBs scale automatically), the rule follows it. Anyone who discovers an
> instance's direct public IP and tries to connect gets silently dropped. All
> traffic must flow through the load balancer.

> **Production Note:** HTTP only (port 80) is acceptable for a learning lab.
> In production, the bank would add HTTPS (port 443) with a TLS certificate
> and redirect all port 80 traffic to 443 automatically. Unencrypted HTTP
> would never be used for real customer banking traffic.

---

### Step 5: Create Launch Template

The launch template is the blueprint every Auto Scaling Group instance is built
from. Every new instance launched by the ASG is an exact copy of this template.

| Setting | Value |
|---|---|
| Name | `levelup-webserver-lt` |
| AMI | Amazon Linux 2023 (free tier eligible, 64-bit x86) |
| Instance type | `t2.micro` |
| Key pair | None (traffic routes through ALB — SSH not required) |
| Security group | `levelup-webserver-sg` |

**User data script:**

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Level-Up Bank | Instance: $(hostname -f)</h1>" > /var/www/html/index.html
```

> **Two most important lines:**
> - **`systemctl enable httpd`** — ensures Apache starts automatically on
>   every reboot. Without this, a rebooted instance serves no traffic but
>   still passes the EC2 status check, creating a silent failure the ALB
>   health check would eventually catch
> - **`$(hostname -f)`** — injects the instance's own hostname into the page
>   at launch. When multiple instances are running, refreshing the ALB URL will
>   show different hostnames — proving the load balancer is distributing traffic
>   across availability zones

---

### Step 6: Create Application Load Balancer and Target Group

**Target Group (created first — selected during ALB setup):**

| Setting | Value |
|---|---|
| Name | `levelup-bank-tg` |
| Target type | Instances |
| Protocol | HTTP |
| Port | 80 |
| VPC | `levelup-bank-vpc` |
| Health check path | `/` |
| Registered targets | None — ASG registers instances automatically |

**Application Load Balancer:**

| Setting | Value |
|---|---|
| Name | `levelup-bank-alb` |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | `levelup-bank-vpc` |
| Availability Zones | `levelup-public-1a`, `levelup-public-1b`, `levelup-public-1c` |
| Security group | `levelup-alb-sg` |
| Listener | HTTP:80 → Forward to `levelup-bank-tg` (100%) |
| DNS name | `levelup-bank-alb-1256661185.us-east-1.elb.amazonaws.com` |

> **Two most important settings:**
> - **Internet-facing scheme** — gives the ALB a public DNS name reachable
>   from anywhere on the internet. An internal ALB would only be reachable
>   from inside the VPC — correct for internal services, wrong here
> - **No registered targets** — this is intentional. The Auto Scaling Group
>   registers and deregisters instances automatically as it scales up and down.
>   Manually registering targets would conflict with ASG management

> **Cost Note:** ALBs are not free tier eligible. In a real AWS account an ALB
> costs at least $16/month at minimum plus Load Balancer Capacity Units (LCUs) costs.
> Always delete the ALB when the lab is complete.

---

### Step 7: Create Auto Scaling Group

| Setting | Value |
|---|---|
| Name | `levelup-bank-asg` |
| Launch template | `levelup-webserver-lt` (Default version) |
| VPC | `levelup-bank-vpc` |
| Subnets | `levelup-public-1a`, `levelup-public-1b`, `levelup-public-1c` |
| Load balancing | Attach to existing target group: `levelup-bank-tg` |
| Health check type | ELB (not EC2 only) |
| Desired capacity | 2 |
| Minimum capacity | 2 |
| Maximum capacity | 5 |
| Scaling policies | None (added in Advanced tier) |
| Instance name tag | `levelup-bank-webserver` |

> **Two most important settings:**
> - **ELB health checks** — the ASG asks the load balancer whether each
>   instance is healthy, not just whether the EC2 is running. If Apache
>   crashes but the instance stays on, EC2-only health checks miss it.
>   ELB health checks catch it because the HTTP request to `/` fails
> - **Desired = Minimum = 2** — the ASG immediately launches 2 instances on
>   creation and will never scale below 2, even during idle periods. This is
>   the high availability floor — one instance per AZ at all times

---

### Step 8: Verify

Navigated to the ALB DNS URL in a browser:
`http://levelup-bank-alb-1256661185.us-east-1.elb.amazonaws.com`

**Browser verification — ALB routing:**
- Confirmed page: **"Level-Up Bank | Instance: ip-10-10-1-206.ec2.internal"**
- Refreshed — confirmed different instance: **"Level-Up Bank | Instance: ip-10-10-3-9.ec2.internal"**
- Two different hostnames from two different AZs confirm load balancing is active

**Direct IP block verification:**
- Navigated to the EC2 instance's public IP directly: `http://3.84.249.127`
- Confirmed: `ERR_CONNECTION_TIMED_OUT` — `levelup-webserver-sg` is blocking
  all traffic that did not originate from `levelup-alb-sg`

---

## Troubleshooting Notes

### Issue: 502 Bad Gateway on ALB URL

**Symptom:** ALB DNS returns `502 Bad Gateway`. ALB is Active, instances are
Running with 2/2 status checks, but target group shows instances as Unhealthy.

**Root cause:** Subnets had **Auto-assign public IPv4** disabled (the default
on custom VPCs). Without a public IP, instances could not reach the internet
during boot to run `yum install -y httpd`. Apache never installed. Port 80
never opened. ALB health checks failed.

**Fix:**
1. VPC → Subnets → select each subnet → Actions → Edit subnet settings →
   Enable **Auto-assign public IPv4 address** → Save (repeat for all 3)
2. EC2 → Auto Scaling Groups → `levelup-bank-asg` → Instance refresh →
   Start instance refresh (replaces broken instances with correctly configured ones)

> **Key lesson:** Custom VPCs default to Auto-assign public IPv4 OFF — the
> stricter and more correct production default. The AWS default VPC has this
> ON, which is why it's easy to miss when building from scratch. In a real
> production environment the correct fix is a **NAT Gateway**, which lets
> instances call out to the internet for updates without exposing a public
> IP on each instance.

---

### Issue: Wrong CIDR on VPC

**Symptom:** VPC created with `10.0.0.0/16` instead of `10.10.0.0/16`.

**Root cause:** Typo during VPC creation.

**Fix:** Delete the incorrect VPC and recreate. CIDR blocks cannot be edited
after a VPC is created. Any subnets built on top of a wrong CIDR would also
need to be deleted.

> **Key lesson:** Verify the CIDR immediately after VPC creation before
> building any dependent resources. One wrong digit cascades into every
> subnet, route, and security group built on top of it.

---

## Running Resources

| Resource | Name | ID |
|---|---|---|
| VPC | `levelup-bank-vpc` | `vpc-0772da34b1b80bce3` |
| Subnet 1a | `levelup-public-1a` | `subnet-095a5b69853e8d82c` |
| Subnet 1b | `levelup-public-1b` | `subnet-059898334272e581b` |
| Subnet 1c | `levelup-public-1c` | `subnet-05d6a6e37b5ef4325` |
| Internet Gateway | `levelup-bank-igw` | `igw-00f5e627002855343` |
| Route Table | `levelup-public-rt` | `rtb-0bc3ed54d8b0850a1` |
| ALB Security Group | `levelup-alb-sg` | `sg-032b48ef96a56b54c` |
| Web Server Security Group | `levelup-webserver-sg` | `sg-03afe37d7785fcead` |
| Launch Template | `levelup-webserver-lt` | `lt-045efb6d2a48ae4d1` |
| Target Group | `levelup-bank-tg` | `c2b60b6289831612` |
| Load Balancer | `levelup-bank-alb` | `arn:aws:elasticloadbalancing:us-east-1:992382750898:loadbalancer/app/levelup-bank-alb/cc2d3f3bb905b652` |
| Auto Scaling Group | `levelup-bank-asg` | min: 2 / desired: 2 / max: 5 |

---

## Advanced Tier — CPU Scaling Policy + Stress Test

> *Coming next — Adding a Target Tracking Scaling Policy at 50% CPU and
> stress testing with the `stress` tool to trigger scale-out.*

---

## Complex Tier — EFS Shared File System

> *Coming next — Adding Amazon EFS so all ASG instances share a common
> file system. Files uploaded to one instance are visible on all others
> automatically.*

---

## Teardown (Cost Management)

When this project is not actively in use, delete resources in this order to
avoid unnecessary charges:

```bash
# 1. Delete the Auto Scaling Group (terminates all managed instances)
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name levelup-bank-asg \
  --force-delete

# 2. Delete the Application Load Balancer
aws elbv2 delete-load-balancer \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:992382750898:loadbalancer/app/levelup-bank-alb/cc2d3f3bb905b652

# 3. Delete the Target Group
aws elbv2 delete-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:992382750898:targetgroup/levelup-bank-tg/c2b60b6289831612

# 4. Delete the Launch Template
aws ec2 delete-launch-template \
  --launch-template-id lt-045efb6d2a48ae4d1

# 5. Delete Security Groups (after ALB and instances are gone)
aws ec2 delete-security-group --group-id sg-032b48ef96a56b54c
aws ec2 delete-security-group --group-id sg-03afe37d7785fcead

# 6. Detach and delete the Internet Gateway
aws ec2 detach-internet-gateway \
  --internet-gateway-id igw-00f5e627002855343 \
  --vpc-id vpc-0772da34b1b80bce3
aws ec2 delete-internet-gateway --internet-gateway-id igw-00f5e627002855343

# 7. Delete Subnets
aws ec2 delete-subnet --subnet-id subnet-095a5b69853e8d82c
aws ec2 delete-subnet --subnet-id subnet-059898334272e581b
aws ec2 delete-subnet --subnet-id subnet-05d6a6e37b5ef4325

# 8. Delete Route Table
aws ec2 delete-route-table --route-table-id rtb-0bc3ed54d8b0850a1

# 9. Delete the VPC
aws ec2 delete-vpc --vpc-id vpc-0772da34b1b80bce3
```

> **Engineer's Note:** Always clean up cloud resources after learning projects.
> The ALB alone costs ~$16/month at minimum. Leaving idle resources running
> is a common source of unexpected AWS bills and is poor cloud hygiene.

---

## Security Findings & Next Steps

| Finding | Risk | Remediation |
|---|---|---|
| HTTP only — no HTTPS | High — customer traffic is unencrypted | Add ACM certificate + HTTPS listener on ALB, redirect port 80 → 443 |
| No WAF on ALB | Medium — vulnerable to SQL injection, XSS | Add AWS WAF with managed rule groups (~$6/month minimum) |
| Auto-assign public IPv4 on subnets | Medium — instances exposed with public IPs | Use private subnets + NAT Gateway in production |
| No VPC Flow Logs | Low — no visibility into rejected traffic | Enable Flow Logs to S3 or CloudWatch for audit trail |
| No HTTPS on health checks | Low — health check traffic unencrypted | Acceptable internally; upgrade with HTTPS listener |

---

## What I Learned

- Custom VPCs are fully isolated by default — an Internet Gateway and Route
  Table are both required before any traffic flows in or out
- A subnet is not "public" just because it exists in a VPC with an IGW —
  it must also be explicitly associated with a route table that has a
  `0.0.0.0/0 → IGW` route
- Security groups can reference other security groups as a traffic source —
  this locks access to a specific identity, not a specific IP, and automatically
  follows ALB IP changes as it scales
- Auto-assign public IPv4 is OFF by default on custom VPCs. Without it,
  instances cannot reach the internet during bootstrapping — `yum install`
  fails silently, Apache never starts, and the ALB returns 502
- ELB health checks are stricter than EC2 health checks — they catch
  application-level failures (Apache down) that EC2 checks miss entirely
- The `local` route in a route table is added automatically by AWS and
  routes all intra-VPC traffic internally — no internet hop required to
  reach an instance in the same VPC
- ALB distributes traffic across instances using the round-robin algorithm
  by default — verified by refreshing the ALB URL and observing different
  hostnames from different availability zones

---

## Next Iterations

- [ ] Add HTTPS listener with ACM certificate on the ALB
- [ ] Advanced: Add CPU Target Tracking Scaling Policy at 50% threshold
- [ ] Advanced: Stress test with `stress` tool and observe scale-out
- [ ] Complex: Add Amazon EFS and mount on all ASG instances via user data
- [ ] Complex: Verify shared file system by uploading a file on one instance
      and viewing it from another
- [ ] Repeat full implementation using AWS CLI

# FreshCart: Secure Two-Tier AWS Architecture

A capstone project that designs, builds, and documents a production-style two-tier cloud architecture on AWS — built for a fictional grocery-delivery startup, FreshCart, that needs its backend to survive a real traffic spike without exposing anything it doesn't have to.

📖 **Full write-up (architecture decisions, tradeoffs, and a debugging story):** [link to your Medium post]

<img width="1365" height="683" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/b17abe80-b707-499f-8ee7-ff48f34dcf62" />


## What this is

A single ordered AWS CLI script (`freshcart-aws-build.sh`) that provisions an entire two-tier network from scratch: a custom VPC, public and private subnets across two Availability Zones, a load-balanced backend tier with no public IP exposure, and a separate static site tier served over HTTPS via CloudFront.

The goal wasn't to build FreshCart's actual application — it's a placeholder `nginx` server standing in for "the backend." The point of this project is the architecture around it: network segmentation, controlled public exposure, health checks, and cost-aware storage lifecycle management.

## Architecture overview

**Public tier**
- Application Load Balancer (ALB), internet-facing, spanning two public subnets
- The *only* public entry point to the backend — nothing else in the private tier is reachable from the internet
- CloudFront distribution serving a static marketing site out of S3, HTTPS-only

**Private tier**
- Two backend EC2 instances (`t3.micro`), one per Availability Zone, each with **no public IP**
- Reachable only from the ALB's security group — not from `0.0.0.0/0`, and not from each other
- Managed exclusively via AWS Systems Manager (SSM) Session Manager — no SSH keys, no open port 22
- NAT Gateway provides outbound-only internet access for OS updates and package installs

**Storage**
- S3 bucket with a lifecycle rule: objects transition to `STANDARD_IA` after 30 days, expire after 90 days

| Component | Service | Notes |
|---|---|---|
| Networking | Custom VPC, 4 subnets (2 public / 2 private), 2 AZs | `10.0.0.0/16` |
| Compute | 2× EC2 `t3.micro`, Amazon Linux 2023 | Private subnets, no public IP |
| Load balancing | Application Load Balancer + Target Group | Health check on `/health` |
| Egress | NAT Gateway + Internet Gateway | Private subnets route outbound only through NAT |
| Static site | S3 + CloudFront | HTTPS via CloudFront default certificate |
| Access management | IAM instance profile + SSM Session Manager | No SSH key pair used |
| Cost control | S3 lifecycle rule (30-day IA transition, 90-day expiration) | |

## Repository contents

```
.
├── freshcart-aws-build.sh      # Full ordered AWS CLI script — provisions everything below
├── Architecture_Diagram.png    # Final annotated architecture diagram
├── FreshCart_Architecture.drawio  # Editable diagram source (draw.io)
└── README.md
```

## Running it yourself

**Prerequisites**
- AWS CLI v2, configured with credentials that have permissions for EC2, ELBv2, S3, IAM, and CloudFront
- An AWS account with the default service quotas (this fits comfortably in the free tier if torn down promptly)
- Bash (Linux, macOS, or Git Bash / WSL on Windows)

**Before you run it**
- This script creates real, billable resources — most notably a NAT Gateway and an Application Load Balancer, both of which bill hourly regardless of traffic. Set a billing alert before running this.
- Update the placeholder values at the top of the script (`BUCKET_NAME`, region, CIDR ranges if needed) — S3 bucket names must be globally unique.
- You'll need `/tmp/index.html` (your static site content) to exist before the S3 upload step runs.

```bash
chmod +x freshcart-aws-build.sh
./freshcart-aws-build.sh
```

The script prints the ALB DNS name and CloudFront domain at the end. Give CloudFront 10–20 minutes to finish deploying before testing HTTPS.

**Verifying it worked:**
```bash
curl http://$ALB_DNS
curl http://$ALB_DNS/health
aws cloudfront get-distribution --id "$CF_DISTRIBUTION_ID" --query 'Distribution.Status' --output text
curl -I https://$CF_DOMAIN
```

**Tearing it down:** this script does not include a teardown — resources should be deleted manually via the console or CLI once you're done reviewing (Load Balancer → Target Group → NAT Gateway → EIP release → subnets/route tables → VPC → S3 bucket → CloudFront distribution, roughly in that order due to dependency ordering).

## Key design decisions

- **No public IP on the backend.** Removing direct internet exposure from the compute layer removes an entire class of attack surface. Management happens entirely through SSM Session Manager instead of SSH.
- **Security groups reference security groups, not CIDR blocks.** The backend's inbound rule allows traffic only from the ALB's security group ID — not from `0.0.0.0/0`, and not even from the rest of the VPC.
- **Two Availability Zones.** Both the backend tier and the public subnets are duplicated across AZs so a single zone failure doesn't take down the whole service.
- **A 30/90-day S3 lifecycle window**, chosen as a reasonable default in the absence of real traffic data — see the full write-up for the reasoning and its limitations.

For the full reasoning behind each of these — including a debugging story about a silently-failing user-data script that took both backend instances offline for hours — see the [linked blog post](#) above.

## Known limitations / what I'd change for production

- The NAT Gateway is single-AZ (in `us-east-1a`), so an outage there would also cut outbound internet access for the `us-east-1b` backend instance. Production would use one NAT Gateway per AZ.
- No Auto Scaling Group — instance count is fixed at two.
- No WAF, no CloudWatch alarms, no centralized logging.
- Infrastructure is defined as an imperative CLI script rather than Terraform/CloudFormation, so it isn't safely re-runnable.

## License

This is a personal learning/capstone project. Feel free to reference or adapt it for your own learning.

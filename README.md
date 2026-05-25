# Fault-Tolerant Web App — CloudFormation Stack

A self-healing, auto-scaling marketing website on AWS. Traffic is distributed across multiple EC2 instances by an Application Load Balancer. An Auto Scaling Group maintains availability and scales on CPU demand — no manual intervention needed.

---

## Architecture Overview

```
Internet
   │
   ▼
Application Load Balancer  (port 80, multi-AZ)
   │
   ▼
Target Group  (HTTP health checks on /)
   │
   ├── EC2 Instance 1  (Apache · Hello World · hostname)
   ├── EC2 Instance 2  (Apache · Hello World · hostname)
   └── EC2 Instance N  (auto-launched on CPU spike)
         ▲
         │
   Launch Template  (AMI + User Data bootstrap script)
         │
   Auto Scaling Group  (min 2 · desired 2 · max 6)
         │
   CloudWatch Alarms  (CPU ≥ 70% → scale out / CPU ≤ 30% → scale in)
```

---

## Prerequisites

| Requirement | Notes |
|---|---|
| AWS CLI ≥ 2.x | `aws --version` |
| IAM permissions | CloudFormation, EC2, ELB, AutoScaling, IAM, CloudWatch |
| Existing VPC | With at least **two public subnets** in different AZs |
| EC2 Key Pair | For SSH access (optional if using SSM Session Manager) |

---

## Quick Start

### 1. Clone / download the template

```bash
git clone <your-repo>
cd fault-tolerant-web-app
```

### 2. Copy and edit the deployment config

```bash
cp deployment-config.yml deployment-config.local.yml
# Edit deployment-config.local.yml with your VPC/subnet IDs
```

### 3. Deploy

```bash
aws cloudformation deploy \
  --template-file fault-tolerant-web-app.yaml \
  --stack-name my-web-app \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    VpcId=vpc-xxxxxxxx \
    PublicSubnetIds=subnet-aaa,subnet-bbb \
    KeyPairName=my-key-pair
```

Or use the config file approach (see [Deployment Config](#deployment-config) below).

### 4. Get the ALB URL

```bash
aws cloudformation describe-stacks \
  --stack-name my-web-app \
  --query "Stacks[0].Outputs[?OutputKey=='LoadBalancerDNS'].OutputValue" \
  --output text
```

Open the URL in your browser. **Refresh repeatedly** — you'll see different hostnames proving the load balancer is routing across instances.

---

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `VpcId` | — | **Required.** VPC where resources are created |
| `PublicSubnetIds` | — | **Required.** ≥2 public subnets in different AZs |
| `InstanceType` | `t3.micro` | EC2 instance size |
| `KeyPairName` | — | **Required.** EC2 key pair name |
| `AsgMinSize` | `2` | Minimum instances (floor) |
| `AsgMaxSize` | `6` | Maximum instances (ceiling) |
| `AsgDesiredCapacity` | `2` | Initial running instance count |
| `CpuScaleOutThreshold` | `70` | CPU % that triggers scale-out |
| `CpuScaleInThreshold` | `30` | CPU % that triggers scale-in |

---

## Deployment Config

`deployment-config.yml` stores your environment-specific values so you don't repeat them on the CLI. See the file for full documentation. Three environments are pre-configured: `dev`, `staging`, `production`.

```bash
# Deploy a specific environment using the config
./scripts/deploy.sh production
```

---

## Testing

### Verify load balancing

```bash
ALB_DNS=$(aws cloudformation describe-stacks \
  --stack-name my-web-app \
  --query "Stacks[0].Outputs[?OutputKey=='LoadBalancerDNS'].OutputValue" \
  --output text)

# Hit the endpoint 10 times — hostnames should rotate
for i in {1..10}; do curl -s $ALB_DNS | grep "Instance ID"; done
```

### Stress test (trigger scale-out)

```bash
# SSH into any instance
ssh -i my-key-pair.pem ec2-user@<instance-public-ip>

# Install stress tool
sudo yum install -y stress

# Spike CPU for 10 minutes
stress --cpu 4 --timeout 600
```

Watch the ASG in the AWS Console → EC2 → Auto Scaling Groups. Within 4–6 minutes a new instance will appear.

### Test self-healing (terminate an instance)

```bash
# Terminate an instance manually
aws ec2 terminate-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

The ASG detects the capacity drop and launches a replacement automatically — typically within 2 minutes.

---

## Cleanup

```bash
aws cloudformation delete-stack --stack-name my-web-app

# Confirm deletion
aws cloudformation wait stack-delete-complete --stack-name my-web-app
```

---

## File Structure

```
.
├── fault-tolerant-web-app.yaml   # Main CloudFormation template
├── deployment-config.yml         # Environment parameter values
└── README.md                     # This file
```

---

## Security Notes

- EC2 instances accept HTTP **only from the ALB security group**, not the open internet.
- The SSH rule (`0.0.0.0/0`) is left open for the challenge. In production, restrict it to your IP or remove it and use **SSM Session Manager** instead (the IAM role already includes `AmazonSSMManagedInstanceCore`).
- EBS volumes are encrypted at rest.
- IMDSv2 is enforced on all instances.
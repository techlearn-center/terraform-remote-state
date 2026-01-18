# Terraform Remote State & AWS Access Management

## Master Remote Backends, IAM, and Secure Server Access

Welcome to the **Terraform Remote State Challenge**! This is an essential skill for any DevOps engineer working with Terraform in a team environment. You'll learn how to:

- Store Terraform state remotely in S3
- Implement state locking with DynamoDB
- Create and manage IAM users and access keys
- Set up SSH key pairs for EC2 access
- Follow security best practices

---

## 🚀 Quick Start

> **GitHub Account Required** for grading and submission (Options A & B).
> Don't have one? [Create a free GitHub account](https://github.com/signup) - it's free and essential for any DevOps career!
>
> Option C works without GitHub but you won't get automated grading.

### Step 1: Get the Code

**Option A: Fork (Easiest)**
```bash
# 1. Click "Fork" button on GitHub (top right of this page)
# 2. Clone YOUR fork:
git clone https://github.com/YOUR-USERNAME/terraform-remote-state.git
cd terraform-remote-state
```

**Option B: Clone & Create Your Own Repo**
```bash
# 1. Clone this repo
git clone https://github.com/techlearn-center/terraform-remote-state.git
cd terraform-remote-state

# 2. Remove the original remote
git remote remove origin

# 3. Create a new repo on GitHub (github.com/new)
# 4. Add your new repo as origin
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO-NAME.git
git push -u origin main
```

**Option C: Clone Only (Practice, No Grading)**
```bash
git clone https://github.com/techlearn-center/terraform-remote-state.git
cd terraform-remote-state
# Note: You won't be able to push or get GitHub Actions grading
```

### Step 2: Choose Your Path

| Path | Best For | Requirements |
|------|----------|--------------|
| **LocalStack** | Learning, no cost | Docker, Terraform |
| **Real AWS** | Production experience | AWS account, Terraform |

### Step 3: Complete the Tasks

```bash
# For LocalStack:
docker-compose up -d          # Start LocalStack
cd backend && terraform init  # Start with backend module

# For Real AWS:
aws configure                 # Set up credentials
cd backend && terraform init  # Start with backend module
```

### Step 4: Check Your Progress

```bash
python run.py                 # Local progress checker
python dashboard.py           # Visual dashboard
```

### Step 5: Submit Your Work

```bash
git add .
git commit -m "Complete terraform-remote-state challenge"
git push origin main
# Check GitHub Actions for your grade!
```

---

## 📚 Table of Contents

1. [Why Remote State?](#why-remote-state)
2. [Understanding Terraform State](#understanding-terraform-state)
3. [S3 Backend Deep Dive](#s3-backend-deep-dive)
4. [State Locking with DynamoDB](#state-locking-with-dynamodb)
5. [IAM Users and Access Keys](#iam-users-and-access-keys)
6. [SSH Key Pairs for EC2](#ssh-key-pairs-for-ec2)
7. [Prerequisites](#prerequisites)
8. [Challenge Overview](#challenge-overview)
9. [Tasks](#tasks)
10. [Security Best Practices](#security-best-practices)
11. [Troubleshooting](#troubleshooting)
12. [Next Steps](#next-steps)

---

## Why Remote State?

### The Problem with Local State

By default, Terraform stores state in a local file called `terraform.tfstate`:

```
my-project/
├── main.tf
├── variables.tf
└── terraform.tfstate  ← Local state file
```

**Problems with local state:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOCAL STATE PROBLEMS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ Team Collaboration                                          │
│     • State file on one person's laptop                         │
│     • Others can't see current infrastructure                   │
│     • Risk of conflicting changes                               │
│                                                                 │
│  ❌ Security Risk                                                │
│     • State contains sensitive data (passwords, keys)           │
│     • Easy to accidentally commit to Git                        │
│     • No encryption by default                                  │
│                                                                 │
│  ❌ No Locking                                                   │
│     • Two people can run terraform apply simultaneously         │
│     • Results in corrupted state or resource conflicts          │
│                                                                 │
│  ❌ Data Loss                                                    │
│     • Laptop crash = lost state                                 │
│     • No versioning or backup                                   │
│     • Terraform loses track of resources                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The Solution: Remote State

```
┌─────────────────────────────────────────────────────────────────┐
│                    REMOTE STATE BENEFITS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Team Collaboration                                          │
│     • Shared state accessible by all team members               │
│     • Everyone sees the same infrastructure state               │
│     • CI/CD pipelines can access state                          │
│                                                                 │
│  ✅ Security                                                     │
│     • State encrypted at rest (S3 encryption)                   │
│     • Access controlled via IAM policies                        │
│     • Never committed to Git                                    │
│                                                                 │
│  ✅ State Locking                                                │
│     • DynamoDB prevents concurrent modifications                │
│     • Only one terraform apply at a time                        │
│     • Prevents state corruption                                 │
│                                                                 │
│  ✅ Durability                                                   │
│     • S3 provides 99.999999999% durability                      │
│     • Automatic versioning for rollback                         │
│     • No single point of failure                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Understanding Terraform State

### What's in the State File?

Terraform state tracks the mapping between your configuration and real resources:

```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [
        {
          "attributes": {
            "id": "i-0abc123def456",
            "ami": "ami-12345678",
            "instance_type": "t2.micro",
            "private_ip": "10.0.1.50",
            "public_ip": "54.123.45.67",
            "tags": {
              "Name": "web-server"
            }
          }
        }
      ]
    }
  ]
}
```

### State Contains Sensitive Data!

⚠️ **Important**: State may contain:
- Database passwords
- API keys
- Private IP addresses
- Resource ARNs
- Connection strings

**Never commit state to Git!**

### The State Workflow

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Your Code      │     │  Terraform CLI   │     │   Remote State   │
│   (main.tf)      │     │                  │     │   (S3 Bucket)    │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │                        │                        │
         │  terraform plan        │                        │
         │───────────────────────▶│                        │
         │                        │    Read state          │
         │                        │───────────────────────▶│
         │                        │◀───────────────────────│
         │                        │                        │
         │                        │    Compare config      │
         │                        │    vs state vs AWS     │
         │                        │                        │
         │  Show plan             │                        │
         │◀───────────────────────│                        │
         │                        │                        │
         │  terraform apply       │                        │
         │───────────────────────▶│                        │
         │                        │    Acquire lock        │
         │                        │───────────────────────▶│ (DynamoDB)
         │                        │                        │
         │                        │    Make changes        │
         │                        │───────────────────────▶│ (AWS)
         │                        │                        │
         │                        │    Update state        │
         │                        │───────────────────────▶│
         │                        │                        │
         │                        │    Release lock        │
         │                        │───────────────────────▶│
         │  Complete              │                        │
         │◀───────────────────────│                        │
```

---

## S3 Backend Deep Dive

### Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### Configuration Options Explained

| Option | Required | Description |
|--------|----------|-------------|
| `bucket` | Yes | S3 bucket name for state storage |
| `key` | Yes | Path within bucket (like a file path) |
| `region` | Yes | AWS region where bucket exists |
| `encrypt` | No | Enable server-side encryption (recommended!) |
| `dynamodb_table` | No | DynamoDB table for state locking |
| `profile` | No | AWS CLI profile to use |
| `role_arn` | No | IAM role to assume |

### Organizing State with Keys

Use meaningful key paths to organize state:

```
my-terraform-state-bucket/
├── prod/
│   ├── network/terraform.tfstate
│   ├── compute/terraform.tfstate
│   └── database/terraform.tfstate
├── staging/
│   ├── network/terraform.tfstate
│   └── compute/terraform.tfstate
└── dev/
    └── all/terraform.tfstate
```

### S3 Bucket Requirements

Your S3 bucket should have:

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-bucket"

  # Prevent accidental deletion
  lifecycle {
    prevent_destroy = true
  }
}

# Enable versioning for state history
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

# Block all public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## State Locking with DynamoDB

### Why Locking Matters

Without locking, this can happen:

```
Time    Developer A              Developer B
─────   ─────────────────────   ─────────────────────
10:00   terraform plan
10:01   (sees 2 instances)      terraform plan
10:02                           (sees 2 instances)
10:03   terraform apply
10:04   (creates instance 3)    terraform apply
10:05                           (also creates instance 3!)
10:06   ❌ CONFLICT! Two instances created with same config
```

With locking:

```
Time    Developer A              Developer B
─────   ─────────────────────   ─────────────────────
10:00   terraform plan
10:01   terraform apply
10:02   🔒 Lock acquired        terraform apply
10:03   (making changes)        ⏳ "Error: state locked"
10:04   (making changes)        (waits or aborts)
10:05   🔓 Lock released
10:06   ✅ Complete             terraform apply
10:07                           🔒 Lock acquired
10:08                           ✅ Proceeds safely
```

### DynamoDB Table Configuration

```hcl
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name    = "Terraform State Lock Table"
    Purpose = "terraform"
  }
}
```

### Lock Table Structure

| LockID | Info | Operation | Who | Created |
|--------|------|-----------|-----|---------|
| `bucket/key` | `{"ID":"xxx","Operation":"apply"}` | OperationApply | user@company.com | 2024-01-15T10:00:00Z |

---

## IAM Users and Access Keys

### IAM Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Account                              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    IAM Users                             │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │  alice  │  │   bob   │  │  carol  │  │ ci-user │    │   │
│  │  │ (Admin) │  │  (Dev)  │  │  (Dev)  │  │(Service)│    │   │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘    │   │
│  └───────┼────────────┼────────────┼────────────┼──────────┘   │
│          │            │            │            │               │
│  ┌───────▼────────────▼────────────▼────────────▼──────────┐   │
│  │                    IAM Groups                            │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────────────┐    │   │
│  │  │  Admins   │  │Developers │  │  ServiceAccounts  │    │   │
│  │  └─────┬─────┘  └─────┬─────┘  └─────────┬─────────┘    │   │
│  └────────┼──────────────┼──────────────────┼───────────────┘   │
│           │              │                  │                   │
│  ┌────────▼──────────────▼──────────────────▼───────────────┐   │
│  │                    IAM Policies                          │   │
│  │  ┌────────────────┐  ┌────────────────────────────────┐ │   │
│  │  │ AdministratorAcc│  │ Custom: TerraformStateAccess  │ │   │
│  │  │ ess            │  │                                │ │   │
│  │  └────────────────┘  └────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Creating IAM Users with Terraform

```hcl
# Create an IAM user
resource "aws_iam_user" "developer" {
  name = "developer"
  path = "/developers/"

  tags = {
    Team = "Platform"
  }
}

# Create access keys for programmatic access
resource "aws_iam_access_key" "developer" {
  user = aws_iam_user.developer.name
}

# Attach a policy to the user
resource "aws_iam_user_policy_attachment" "developer" {
  user       = aws_iam_user.developer.name
  policy_arn = aws_iam_policy.terraform_state.arn
}

# Output the credentials (be careful!)
output "access_key_id" {
  value     = aws_iam_access_key.developer.id
  sensitive = false
}

output "secret_access_key" {
  value     = aws_iam_access_key.developer.secret
  sensitive = true  # Mark as sensitive!
}
```

### Custom Policy for Terraform State Access

```hcl
resource "aws_iam_policy" "terraform_state" {
  name        = "TerraformStateAccess"
  description = "Allow access to Terraform state bucket and lock table"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3StateAccess"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::my-terraform-state-bucket",
          "arn:aws:s3:::my-terraform-state-bucket/*"
        ]
      },
      {
        Sid    = "DynamoDBLockAccess"
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:DeleteItem"
        ]
        Resource = "arn:aws:dynamodb:*:*:table/terraform-state-lock"
      }
    ]
  })
}
```

---

## SSH Key Pairs for EC2

### How SSH Keys Work

```
┌─────────────────────────────────────────────────────────────────┐
│                    SSH KEY AUTHENTICATION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your Computer                          EC2 Instance            │
│  ┌─────────────────┐                   ┌─────────────────┐     │
│  │ Private Key     │                   │ Public Key      │     │
│  │ (id_rsa)        │                   │ (authorized_    │     │
│  │                 │                   │  keys)          │     │
│  │ Keep SECRET!    │                   │ Safe to share   │     │
│  └────────┬────────┘                   └────────┬────────┘     │
│           │                                     │               │
│           │  1. SSH Connection Request          │               │
│           │────────────────────────────────────▶│               │
│           │                                     │               │
│           │  2. Server sends challenge          │               │
│           │◀────────────────────────────────────│               │
│           │                                     │               │
│           │  3. Sign with private key           │               │
│           │────────────────────────────────────▶│               │
│           │                                     │               │
│           │  4. Verify with public key ✅       │               │
│           │◀────────────────────────────────────│               │
│           │                                     │               │
│           │  5. Connection established 🎉       │               │
│           │◀───────────────────────────────────▶│               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Creating SSH Keys with Terraform

#### Option 1: Use Existing Key

```hcl
# Reference an existing key pair
resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("~/.ssh/id_rsa.pub")
}
```

#### Option 2: Generate New Key with Terraform

```hcl
# Generate a new key pair
resource "tls_private_key" "ssh" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

# Create AWS key pair from the generated key
resource "aws_key_pair" "generated" {
  key_name   = "terraform-generated-key"
  public_key = tls_private_key.ssh.public_key_openssh
}

# Save private key locally
resource "local_file" "private_key" {
  content         = tls_private_key.ssh.private_key_pem
  filename        = "${path.module}/private-key.pem"
  file_permission = "0600"
}

# Output for reference
output "private_key_pem" {
  value     = tls_private_key.ssh.private_key_pem
  sensitive = true
}
```

### Using the Key with EC2

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.generated.key_name

  vpc_security_group_ids = [aws_security_group.ssh.id]

  tags = {
    Name = "web-server"
  }
}

# Security group allowing SSH
resource "aws_security_group" "ssh" {
  name        = "allow-ssh"
  description = "Allow SSH inbound traffic"

  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Restrict this in production!
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Connecting to Your Instance

```bash
# Set correct permissions on private key
chmod 600 private-key.pem

# Connect to EC2 instance
ssh -i private-key.pem ec2-user@<public-ip>

# For Ubuntu instances
ssh -i private-key.pem ubuntu@<public-ip>
```

---

## Two Ways to Complete This Challenge

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHOOSE YOUR PATH                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🏠 OPTION 1: LocalStack (Free, Local)                          │
│  ─────────────────────────────────────                          │
│  • No AWS account needed                                        │
│  • No cost                                                      │
│  • Requires Docker                                              │
│  • Great for learning and testing                               │
│  • Resources are simulated locally                              │
│                                                                 │
│  Requirements:                                                  │
│  ✅ Docker and Docker Compose                                   │
│  ✅ Terraform CLI                                               │
│  ✅ Python 3 (for dashboard and progress checker)               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ☁️ OPTION 2: Real AWS (Production Experience)                   │
│  ─────────────────────────────────────────────                  │
│  • Actual AWS resources                                         │
│  • Real-world experience                                        │
│  • May incur small costs (free tier eligible)                   │
│  • NO Docker needed!                                            │
│  • SSH actually works to connect to EC2                         │
│                                                                 │
│  Requirements:                                                  │
│  ✅ AWS Account (free tier works)                               │
│  ✅ AWS CLI installed and configured                            │
│  ✅ Terraform CLI                                               │
│  ❌ Docker NOT required                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Recommendation:** Start with LocalStack to learn, then try real AWS when ready!

---

## Prerequisites

### For LocalStack (Option 1)
- ✅ Docker and Docker Compose installed
- ✅ Terraform CLI installed (v1.0+)
- ✅ Python 3 (for dashboard and run.py)
- ✅ Basic understanding of Terraform

### For Real AWS (Option 2)
- ✅ AWS Account ([Sign up free](https://aws.amazon.com/free/))
- ✅ AWS CLI installed and configured
- ✅ Terraform CLI installed (v1.0+)
- ❌ Docker NOT required

### Helpful Background
- 📖 Completed [terraform-basics](https://github.com/techlearn-center/terraform-basics) challenge
- 📖 Completed [terraform-3tier](https://github.com/techlearn-center/terraform-3tier) challenge
- 📖 Basic understanding of AWS services

### Install Terraform

```bash
# Check if installed
terraform --version

# If not installed, see:
# https://developer.hashicorp.com/terraform/downloads
```

### Install AWS CLI (for Real AWS)

```bash
# Check if installed
aws --version

# If not installed, see:
# https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

# Configure (enter your Access Key ID and Secret)
aws configure
```

---

## Challenge Overview

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHALLENGE ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Part 1: Backend Infrastructure                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  S3 Bucket              DynamoDB Table                   │   │
│  │  (State Storage)        (State Locking)                  │   │
│  │  ┌─────────────┐        ┌─────────────┐                 │   │
│  │  │ terraform-  │        │ terraform-  │                 │   │
│  │  │ state-xxx   │◀──────▶│ state-lock  │                 │   │
│  │  └─────────────┘        └─────────────┘                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Part 2: IAM Configuration                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  IAM User         Access Key        Policy               │   │
│  │  ┌─────────┐     ┌───────────┐     ┌─────────────────┐  │   │
│  │  │terraform│────▶│ AKIA...   │────▶│TerraformState   │  │   │
│  │  │-deployer│     │ secret... │     │Access Policy    │  │   │
│  │  └─────────┘     └───────────┘     └─────────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Part 3: Compute with SSH                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Key Pair           EC2 Instance      Security Group     │   │
│  │  ┌─────────┐       ┌─────────────┐   ┌───────────────┐  │   │
│  │  │ SSH Key │──────▶│ Web Server  │◀──│ Allow SSH     │  │   │
│  │  │ Pair    │       │ (t2.micro)  │   │ (port 22)     │  │   │
│  │  └─────────┘       └─────────────┘   └───────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Directory Structure

```
terraform-remote-state/
├── docker-compose.yml       # LocalStack for local testing
├── README.md               # This file
├── backend/                # Part 1: S3 & DynamoDB setup
│   ├── main.tf            # Provider and backend resources
│   ├── s3.tf              # S3 bucket configuration (TODO)
│   ├── dynamodb.tf        # DynamoDB table (TODO)
│   ├── variables.tf       # Input variables
│   └── outputs.tf         # Output values
├── iam/                   # Part 2: IAM configuration
│   ├── main.tf            # Provider configuration
│   ├── users.tf           # IAM users (TODO)
│   ├── policies.tf        # IAM policies (TODO)
│   ├── variables.tf       # Input variables
│   └── outputs.tf         # Output values
├── compute/               # Part 3: EC2 with SSH
│   ├── main.tf            # Provider with remote backend
│   ├── ssh-key.tf         # SSH key pair (TODO)
│   ├── ec2.tf             # EC2 instance (TODO)
│   ├── security.tf        # Security groups (TODO)
│   ├── variables.tf       # Input variables
│   └── outputs.tf         # Output values
├── solutions/             # Complete solutions
└── run.py                 # Progress checker
```

---

## Tasks

### Task 1: Create S3 Backend Infrastructure

Navigate to the `backend/` directory and complete the S3 and DynamoDB configuration.

**Files to complete:**
- `backend/s3.tf` - S3 bucket with versioning and encryption
- `backend/dynamodb.tf` - DynamoDB table for locking

**Requirements:**
1. Create an S3 bucket with a unique name
2. Enable versioning on the bucket
3. Enable server-side encryption
4. Block all public access
5. Create a DynamoDB table with `LockID` as the hash key

```bash
cd backend
terraform init
terraform plan
terraform apply
```

---

### Task 2: Create IAM User and Access Keys

Navigate to the `iam/` directory and create an IAM user for Terraform.

**Files to complete:**
- `iam/users.tf` - IAM user and access keys
- `iam/policies.tf` - Custom policy for state access

**Requirements:**
1. Create an IAM user named `terraform-deployer`
2. Create access keys for the user
3. Create a custom policy allowing S3 and DynamoDB access
4. Attach the policy to the user

```bash
cd iam
terraform init
terraform plan
terraform apply
```

**Important:** Note the access key ID and secret - you'll need them!

---

### Task 3: Configure Remote Backend and Create EC2

Navigate to the `compute/` directory. This uses the remote backend!

**Files to complete:**
- `compute/main.tf` - Configure the S3 backend
- `compute/ssh-key.tf` - Generate SSH key pair
- `compute/ec2.tf` - Create EC2 instance
- `compute/security.tf` - Security group for SSH

**Requirements:**
1. Configure the S3 backend using values from Task 1
2. Generate an SSH key pair using `tls_private_key`
3. Create an EC2 instance using the key pair
4. Create a security group allowing SSH access
5. Output the connection command

```bash
cd compute
terraform init  # This will configure the remote backend!
terraform plan
terraform apply
```

---

### Task 4: Connect to Your Instance

After applying, use the output to connect:

```bash
# Get the SSH command from outputs
terraform output -raw ssh_command

# Or manually:
ssh -i private-key.pem ec2-user@<public-ip>
```

---

## Getting Started

### Step 1: Start LocalStack

```bash
# Start the local AWS environment
docker-compose up -d

# Verify it's running
docker-compose ps
```

### Step 2: Set Environment Variables

```bash
# For LocalStack
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
```

### Step 3: Complete Each Task in Order

1. Backend first (creates S3 and DynamoDB)
2. IAM second (creates user and keys)
3. Compute last (uses remote backend)

### Step 4: Check Your Progress

```bash
python run.py
```

### Step 5: View the Dashboard

```bash
# Start the visual dashboard
python dashboard.py

# Opens at http://localhost:8080
# Shows your S3, DynamoDB, IAM, EC2 resources
```

---

## Deploying to Real AWS

Want to deploy to actual AWS instead of LocalStack? Here's how:

> **Note:** Docker and LocalStack are **NOT required** for real AWS deployment!
> They are only used for free local testing. For real AWS, you just need:
> - Terraform installed
> - AWS CLI installed
> - An AWS account with credentials

### Step 1: Get AWS Credentials

1. Log in to [AWS Console](https://console.aws.amazon.com)
2. Go to IAM → Users → Create User
3. Attach `AdministratorAccess` policy (for learning only!)
4. Create Access Key → Download credentials

### Step 2: Configure AWS CLI

```bash
# Install AWS CLI if not installed
# https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

# Configure credentials
aws configure
# Enter your Access Key ID
# Enter your Secret Access Key
# Default region: us-east-1
# Default output: json
```

### Step 3: Modify Terraform Files for Real AWS

In each module's `main.tf`, **remove or comment out** the LocalStack configuration:

```hcl
provider "aws" {
  region = var.aws_region

  # REMOVE THESE LINES for real AWS:
  # access_key = "test"
  # secret_key = "test"
  # endpoints { ... }
  # skip_credentials_validation = true
  # skip_metadata_api_check     = true
  # s3_use_path_style           = true
}
```

Your `main.tf` for real AWS should look like:

```hcl
provider "aws" {
  region = var.aws_region
  # AWS CLI credentials will be used automatically
}
```

### Step 4: Deploy!

```bash
cd backend
terraform init
terraform apply

cd ../iam
terraform init
terraform apply

cd ../compute
terraform init
terraform apply
```

### Step 5: Clean Up (Important!)

To avoid AWS charges:

```bash
cd compute && terraform destroy -auto-approve
cd ../iam && terraform destroy -auto-approve
cd ../backend && terraform destroy -auto-approve
```

---

## How Grading Works

Your challenge is graded in **two ways**:

### 1. Local Progress Checker (`run.py`)

Run anytime to check your progress:

```bash
python run.py
```

**Output Example:**
```
============================================================
              CHALLENGE PROGRESS SUMMARY
============================================================

  Total Checks: 21
  Passed: 21
  Failed: 0

  Progress: 100.0%

  [========================================]

  Congratulations! All checks passed!
```

### 2. GitHub Actions (Automated Grading)

When you push to GitHub, the workflow automatically grades your work:

**What it checks:**
| Component | Points | Requirements |
|-----------|--------|--------------|
| S3 Bucket | 4 | Bucket, versioning, encryption, public block |
| DynamoDB | 2 | Table resource, LockID hash key |
| IAM User | 3 | User, access key, policy attachment |
| IAM Policy | 3 | Policy resource, S3 perms, DynamoDB perms |
| SSH Keys | 3 | TLS key, AWS key pair, local file |
| Security Group | 2 | SG resource, port 22 |
| EC2 Instance | 4 | Instance, AMI data, key_name, security group |
| **Total** | **21** | **75% needed to pass** |

**To see your grade:**
1. Push your code to GitHub
2. Go to your repo → Actions tab
3. Click on the latest workflow run
4. View the "Calculate Total Score" step

**Sample Grade Output:**
```
========================================
       CHALLENGE GRADING SUMMARY
========================================

S3 Bucket:      4/4
DynamoDB:       2/2
IAM User:       3/3
IAM Policy:     3/3
SSH Keys:       3/3
Security Group: 2/2
EC2 Instance:   4/4

========================================
TOTAL SCORE: 21/21 (100%)
========================================

Congratulations! All checks passed!
```

---

## Using the Dashboard

The dashboard provides a visual way to see your AWS resources:

```bash
python dashboard.py
```

### Features:
- **S3 Buckets** - Shows your state bucket with versioning status
- **DynamoDB Tables** - Shows your lock table with hash key
- **IAM Users** - Shows users and their access keys
- **SSH Key Pairs** - Shows registered key pairs
- **EC2 Instances** - Shows instances with IPs and state
- **Security Groups** - Shows firewall rules

### Click any section to learn more!
Each section has educational explanations about the resource type.

### Dashboard for Real AWS:
```bash
python dashboard.py --aws
```
This uses your AWS CLI credentials to show real AWS resources.

---

## Security Best Practices

### 1. State File Security

```hcl
# ✅ DO: Enable encryption
terraform {
  backend "s3" {
    encrypt = true
  }
}

# ✅ DO: Use KMS for encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform.arn
    }
  }
}
```

### 2. Access Control

```hcl
# ✅ DO: Block public access
resource "aws_s3_bucket_public_access_block" "state" {
  bucket = aws_s3_bucket.state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ✅ DO: Use least privilege policies
# Only grant the permissions actually needed
```

### 3. Access Keys

```bash
# ❌ DON'T: Hardcode credentials
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."

# ✅ DO: Use AWS profiles or IAM roles
aws configure --profile terraform
export AWS_PROFILE=terraform

# ✅ DO: Rotate keys regularly
# Create new keys before deleting old ones
```

### 4. SSH Keys

```bash
# ✅ DO: Set correct permissions
chmod 600 private-key.pem

# ✅ DO: Use ed25519 for new keys (more secure)
resource "tls_private_key" "ssh" {
  algorithm = "ED25519"
}

# ❌ DON'T: Commit private keys to Git
# Add to .gitignore:
*.pem
*-key.pem
```

### 5. Security Group Rules

```hcl
# ❌ DON'T: Allow SSH from anywhere in production
cidr_blocks = ["0.0.0.0/0"]

# ✅ DO: Restrict to known IPs
cidr_blocks = ["203.0.113.50/32"]  # Your IP only

# ✅ DO: Use a bastion host
# Only the bastion allows SSH from internet
# Other instances only allow SSH from bastion
```

---

## Troubleshooting

### Backend Configuration Errors

```bash
# Error: Backend configuration changed
terraform init -reconfigure

# Error: State lock
terraform force-unlock <LOCK_ID>

# Error: Access denied to S3
# Check IAM permissions and bucket policy
```

### SSH Connection Issues

```bash
# Permission denied
chmod 600 private-key.pem

# Connection refused
# Check security group allows port 22
# Check instance is running
# Check public IP is correct

# Timeout
# Check instance has public IP
# Check internet gateway exists
# Check route table has 0.0.0.0/0 route
```

### LocalStack Issues

```bash
# Services not responding
docker-compose restart

# Check logs
docker-compose logs localstack

# Reset everything
docker-compose down -v
docker-compose up -d
```

---

## Next Steps

After completing this challenge:

1. **Multi-Environment Setup**
   - Use workspaces or directory structure for dev/staging/prod

2. **State Migration**
   - Learn to migrate from local to remote state
   - Practice moving state between backends

3. **CI/CD Integration**
   - Set up GitHub Actions to run Terraform
   - Use the IAM user you created

4. **Advanced IAM**
   - Explore IAM roles and assume role
   - Set up cross-account access

5. **Other Challenges**
   - [monitoring-stack](https://github.com/techlearn-center/monitoring-stack)
   - [logging-stack](https://github.com/techlearn-center/logging-stack)
   - [distributed-tracing](https://github.com/techlearn-center/distributed-tracing)

---

## Resources

- [Terraform S3 Backend Documentation](https://developer.hashicorp.com/terraform/language/settings/backends/s3)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Terraform State Management](https://developer.hashicorp.com/terraform/language/state)
- [AWS Key Pairs Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)

---

## License

This challenge is part of the TechLearn Center curriculum.
Free to use for educational purposes.

---

**Happy Terraforming! 🏗️**

*Remember: Secure state = Secure infrastructure*

# Generate Phase: Infrastructure Artifact Generation

> Loaded by generate.md when generation-infra.json and aws-design.json exist.

**Execute ALL steps in order. Do not skip or optimize.**

## Overview

Transform the design (`aws-design.json`) and migration plan (`generation-infra.json`) into deployable Terraform configurations.

Migration scripts are generated separately by `generate-artifacts-scripts.md` (loaded after this file completes).

**Outputs:**

- `terraform/` directory — Terraform configurations for AWS infrastructure

## Prerequisites

Read the following artifacts from `$MIGRATION_DIR/`:

- `aws-design.json` (REQUIRED) — AWS architecture design with cluster-level resource mappings
- `generation-infra.json` (REQUIRED) — Migration plan with timeline and service assignments
- `preferences.json` (REQUIRED) — User preferences including target region, sizing, compliance
- `gcp-resource-clusters.json` (REQUIRED) — Cluster dependency graph for ordering

Reference files (read as needed during generation):

- `references/design-refs/index.md` — Lookup table: GCP type to design-ref file
- `references/design-refs/compute.md` — Compute service mappings and configurations
- `references/design-refs/database.md` — Database service mappings and configurations
- `references/design-refs/storage.md` — Storage service mappings and configurations
- `references/design-refs/networking.md` — Networking service mappings and configurations
- `references/design-refs/messaging.md` — Messaging service mappings and configurations
- `references/design-refs/ai.md` — AI service mappings and configurations

If any REQUIRED file is missing: **STOP**. Output: "Missing required artifact: [filename]. Complete the prior phase that produces it."

## Output Structure

```
$MIGRATION_DIR/
├── terraform/
│   ├── main.tf              # Provider configuration, backend, data sources
│   ├── variables.tf         # All input variables with types and defaults
│   ├── outputs.tf           # Resource outputs and migration summary
│   ├── vpc.tf               # Networking (VPC, subnets, NAT, security groups)
│   ├── security.tf          # IAM roles, policies, KMS keys
│   ├── storage.tf           # S3 buckets, EFS, backup vaults
│   ├── database.tf          # RDS/Aurora instances, parameter groups
│   ├── compute.tf           # Fargate/ECS, Lambda, EC2
│   └── monitoring.tf        # CloudWatch dashboards, alarms, log groups
```

Not every `.tf` file is needed — only generate files for domains that have resources in `aws-design.json`.

## Step 0: Plan Generation Scope

Before generating any files, build a generation manifest:

1. **Collect resource targets**: Read all resources from `aws-design.json` clusters
2. **Assign to .tf files**: Map each resource to its target .tf file based on `aws_service`:

| AWS Service                                           | Target .tf File | Domain     |
| ----------------------------------------------------- | --------------- | ---------- |
| VPC, Subnet, NAT Gateway, Security Group, Route Table | `vpc.tf`        | networking |
| IAM Role, IAM Policy, KMS Key                         | `security.tf`   | security   |
| S3, EFS, Backup Vault                                 | `storage.tf`    | storage    |
| RDS, Aurora, DynamoDB, ElastiCache                    | `database.tf`   | database   |
| Fargate, ECS, Lambda, EC2                             | `compute.tf`    | compute    |
| CloudWatch, SNS (for alarms)                          | `monitoring.tf` | monitoring |

1. **Build generation manifest**: List which .tf files to generate and which resources go in each

## Step 1: Generate main.tf

### Terraform Block

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # TODO: Configure remote backend for team collaboration
  # backend "s3" {
  #   bucket         = "your-terraform-state-bucket"
  #   key            = "migration/terraform.tfstate"
  #   region         = "us-east-1"
  #   dynamodb_table = "terraform-locks"
  #   encrypt        = true
  # }
}
```

### Provider Block

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
      MigrationId = var.migration_id
    }
  }
}
```

### Data Sources

Add data sources for commonly referenced resources:

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
data "aws_availability_zones" "available" {
  state = "available"
}
```

## Step 2: Generate variables.tf

### Global Variables

Always include these variables:

```hcl
variable "aws_region" {
  description = "AWS region for deployment"
  type        = string
  default     = "us-east-1"  # From preferences.json target_region
}

variable "project_name" {
  description = "Project name for resource naming and tagging"
  type        = string
  default     = "gcp-migration"  # TODO: Set to your project name
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"  # From preferences.json; "dev" unless user specified "production"
}

variable "migration_id" {
  description = "Migration session identifier"
  type        = string
  default     = ""  # From .phase-status.json migration_id
}
```

### Per-Cluster Variables

For each cluster in `aws-design.json`, extract configurable values from `aws_config`:

- **Type inference**: Use `string`, `number`, `bool`, `list(string)`, or `map(string)` based on the value
- **Default values**: Use values from `aws-design.json` `aws_config`
- **Deduplication**: If multiple resources share the same variable (e.g., `aws_region`), define it once

Example for a database cluster:

```hcl
variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t4g.micro"  # From aws-design.json aws_config.instance_class
}

variable "db_allocated_storage" {
  description = "Database storage in GB"
  type        = number
  default     = 20  # From aws-design.json aws_config or source_config
}
```

### Source Config Rules

When `aws-design.json` resources include `gcp_config` (source configuration from GCP):

- Extract configurable values and map to AWS equivalents
- Use GCP values as comments showing the source: `# GCP source: db-custom-2-7680`
- Set AWS defaults based on the mapped configuration

## Step 3: Generate Per-Domain .tf Files

For each domain that has resources in the generation manifest:

### General Rules

1. **Load design references**: For each resource type, consult the appropriate `references/design-refs/*.md` file for AWS configuration best practices
2. **1:Many Expansion**: A single GCP resource may map to multiple AWS resources (e.g., Cloud SQL maps to RDS instance + subnet group + parameter group + security group)
3. **Source Config**: Use `gcp_config` values from `aws-design.json` to populate AWS resource attributes
4. **Inferred mappings**: For resources with `confidence: "inferred"`, add a comment: `# Inferred mapping — verify configuration`
5. **Secondary resources**: Include supporting resources from the cluster's `secondary_resources` (IAM roles, security groups, etc.)
6. **Tags**: Every resource gets standard tags (Project, Environment, ManagedBy, MigrationId) plus domain-specific tags

### vpc.tf — Networking

Generate VPC infrastructure based on networking clusters:

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr  # TODO: Determine appropriate CIDR
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${var.project_name}-vpc" }
}

# Subnets (public + private per AZ)
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = { Name = "${var.project_name}-public-${count.index}" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = { Name = "${var.project_name}-private-${count.index}" }
}

# Internet Gateway, NAT Gateway, Route Tables...
```

### security.tf — IAM and Encryption

Generate IAM roles and policies for each service:

- Fargate tasks need execution role + task role
- RDS needs enhanced monitoring role (if monitoring enabled)
- Lambda needs execution role with appropriate policies
- KMS keys for encryption at rest

**Critical**: Follow least-privilege principle. No `"Action": "*"` or `"Resource": "*"` policies.

### storage.tf — S3 and EFS

For each storage resource in `aws-design.json`:

- S3 buckets with versioning enabled
- Server-side encryption (SSE-S3 or SSE-KMS)
- Lifecycle policies based on storage class mappings
- Block public access by default

### database.tf — RDS, Aurora, DynamoDB

For each database resource in `aws-design.json`:

- Instance with configuration from `aws_config`
- Subnet group using private subnets
- Parameter group with engine-specific settings
- Security group allowing access from compute resources only
- Automated backups enabled
- Encryption at rest

### compute.tf — Fargate, Lambda, EC2

For each compute resource in `aws-design.json`:

- ECS cluster and Fargate service definitions
- Task definitions with CPU/memory from `aws_config`
- Load balancer target groups and listeners
- Auto-scaling policies

### monitoring.tf — CloudWatch

Generate monitoring infrastructure:

- CloudWatch log groups for each service
- Dashboard with key metrics
- Alarms for critical thresholds (from `generation-infra.json` success_metrics)
- SNS topic for alarm notifications

### Domain-Specific Rules

- **VPC**: Always create at least 2 AZs for high availability
- **Security**: IAM policies use specific resource ARNs, never wildcards
- **Storage**: S3 buckets have unique names (use project_name prefix + random suffix)
- **Database**: Multi-AZ follows `preferences.json` availability setting
- **Compute**: Fargate uses private subnets with NAT gateway for internet access
- **Monitoring**: Retention period for logs defaults to 30 days

## Step 4: Generate outputs.tf

### Individual Resource Outputs

For key resources, output identifiers needed for migration scripts and documentation:

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "database_endpoint" {
  description = "RDS/Aurora endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = false
}

output "ecs_cluster_name" {
  description = "ECS cluster name"
  value       = aws_ecs_cluster.main.name
}
```

### Migration Summary Output

```hcl
output "migration_summary" {
  description = "Summary of migrated resources"
  value = {
    region          = var.aws_region
    vpc_id          = aws_vpc.main.id
    environment     = var.environment
    services_count  = length(local.all_services)
    migration_id    = var.migration_id
  }
}
```

## Step 5: Self-Check

After generating all terraform files, verify the following quality rules:

### Terraform Quality Rules

1. **No wildcard IAM policies**: No `"Action": "*"` or `"Resource": "*"` in any IAM policy
2. **No default VPC**: All resources reference the created VPC, not the default VPC
3. **No hardcoded credentials**: No AWS access keys, secret keys, or passwords in .tf files
4. **Tags everywhere**: Every resource has at least Project, Environment, ManagedBy, MigrationId tags
5. **Encryption enabled**: All storage (S3, EBS, RDS) has encryption at rest
6. **Private subnets for data**: Databases and internal services use private subnets
7. **Security groups are specific**: No `0.0.0.0/0` ingress rules except for ALB port 443
8. **Variables are typed**: Every variable has a `type` and `description`
9. **Outputs are documented**: Every output has a `description`
10. **No hardcoded regions**: Region comes from `var.aws_region`, not hardcoded strings

## Phase Completion

Report the list of generated terraform files to the parent orchestrator. **Do NOT update `.phase-status.json`** — the parent `generate.md` handles phase completion.

Output:

```
Generated terraform artifacts:
- terraform/main.tf
- terraform/variables.tf
- terraform/outputs.tf
- terraform/[domain].tf (for each domain with resources)

Total: [N] Terraform files
TODO markers: [N] items requiring manual configuration
```

## Example: Networking Cluster Generation

Given a networking cluster in `aws-design.json`:

```json
{
  "cluster_id": "networking_vpc_us-central1_001",
  "resources": [
    {
      "gcp_address": "google_compute_network.main",
      "gcp_type": "google_compute_network",
      "aws_service": "VPC",
      "aws_config": { "cidr_block": "10.0.0.0/16", "region": "us-east-1" }
    },
    {
      "gcp_address": "google_compute_subnetwork.app",
      "gcp_type": "google_compute_subnetwork",
      "aws_service": "Subnet",
      "aws_config": { "cidr_block": "10.0.1.0/24", "availability_zone": "us-east-1a" }
    }
  ]
}
```

This generates in `vpc.tf`:

```hcl
# Networking cluster: networking_vpc_us-central1_001
# Source: google_compute_network.main → AWS VPC

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr  # GCP source: 10.0.0.0/16
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${var.project_name}-vpc" }
}

# Source: google_compute_subnetwork.app → AWS Subnet
resource "aws_subnet" "app" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"  # GCP source: google_compute_subnetwork.app
  availability_zone = "us-east-1a"

  tags = { Name = "${var.project_name}-app" }
}
```

And in `variables.tf`:

```hcl
variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"  # From GCP google_compute_network.main
}
```

# Generate Phase: Infrastructure Artifact Generation

> Loaded by generate.md when generation-infra.json and aws-design.json exist.

**Execute ALL steps in order. Do not skip or optimize.**

## Overview

Transform the design (`aws-design.json`) and migration plan (`generation-infra.json`) into deployable Terraform configurations and migration scripts.

**Outputs:**

- `terraform/` directory — Terraform configurations for AWS infrastructure
- `scripts/` directory — Numbered migration scripts for data and service migration

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
├── scripts/
│   ├── 01-validate-prerequisites.sh
│   ├── 02-migrate-data.sh
│   ├── 03-migrate-containers.sh
│   ├── 04-migrate-secrets.sh
│   └── 05-validate-migration.sh
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
1. **Identify scripts needed**: Based on services requiring data migration, container migration, or secrets migration

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

## Step 5: Generate Migration Scripts

### Script Rules

- Every script defaults to **dry-run mode** — requires `--execute` flag to make changes
- Every script includes a verification step after execution
- Scripts are numbered for execution order
- Scripts use `set -euo pipefail` for safety
- Scripts log all actions to `$MIGRATION_DIR/logs/`

### 01-validate-prerequisites.sh

Verify all prerequisites before migration:

- AWS CLI configured and authenticated
- Required IAM permissions present
- Target VPC and subnets exist (Terraform applied)
- GCP connectivity established (for data transfer)
- Required tools installed (aws, gcloud, terraform, jq)

### 02-migrate-data.sh

Based on database resources in `aws-design.json`:

**Cloud SQL to RDS/Aurora:**

```bash
#!/usr/bin/env bash
set -euo pipefail

# Cloud SQL → RDS data migration
# Usage: ./02-migrate-data.sh [--execute]

DRY_RUN=true
[[ "${1:-}" == "--execute" ]] && DRY_RUN=false

echo "=== Database Migration: Cloud SQL → RDS ==="
echo "Mode: $([ "$DRY_RUN" = true ] && echo 'DRY RUN' || echo 'EXECUTE')"

# TODO: Configure source and target connection details
SOURCE_HOST="<cloud-sql-ip>"      # TODO: Set Cloud SQL IP
TARGET_HOST="<rds-endpoint>"       # From terraform output database_endpoint
DATABASE_NAME="<database>"         # TODO: Set database name

if [ "$DRY_RUN" = true ]; then
  echo "[DRY RUN] Would export from Cloud SQL: $SOURCE_HOST"
  echo "[DRY RUN] Would import to RDS: $TARGET_HOST"
  echo "[DRY RUN] Database: $DATABASE_NAME"
else
  # Export from Cloud SQL
  echo "Exporting from Cloud SQL..."
  gcloud sql export sql "$SOURCE_HOST" "gs://migration-bucket/export.sql" \
    --database="$DATABASE_NAME"

  # Import to RDS
  echo "Importing to RDS..."
  # TODO: Use pg_restore, mysql, or appropriate tool for your database engine
  # psql -h "$TARGET_HOST" -U admin -d "$DATABASE_NAME" < export.sql
fi

# Verification
echo "=== Verification ==="
echo "TODO: Compare row counts between source and target"
echo "TODO: Run checksum validation on critical tables"
```

**BigQuery to S3 (if applicable):**

```bash
# BigQuery → S3 data export
# TODO: Configure BigQuery dataset and S3 bucket
# bq extract --destination_format=PARQUET 'dataset.table' 'gs://bucket/export/'
# aws s3 sync gs://bucket/export/ s3://target-bucket/import/
```

**Firestore to DynamoDB (if applicable):**

```bash
# Firestore → DynamoDB migration
# TODO: Use AWS DMS or custom export/import script
```

### 03-migrate-containers.sh

Migrate container images from GCR/Artifact Registry to ECR:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Container image migration: GCR → ECR
# Usage: ./03-migrate-containers.sh [--execute]

DRY_RUN=true
[[ "${1:-}" == "--execute" ]] && DRY_RUN=false

AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION="us-east-1"  # From preferences.json target_region
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

# TODO: List container images from aws-design.json compute resources
IMAGES=(
  "gcr.io/project/image1:latest"
  # Add more images from your GCP container registry
)

for IMAGE in "${IMAGES[@]}"; do
  IMAGE_NAME=$(echo "$IMAGE" | rev | cut -d'/' -f1 | rev | cut -d':' -f1)
  IMAGE_TAG=$(echo "$IMAGE" | rev | cut -d':' -f1 | rev)

  if [ "$DRY_RUN" = true ]; then
    echo "[DRY RUN] Would migrate: $IMAGE → $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG"
  else
    echo "Creating ECR repository: $IMAGE_NAME"
    aws ecr create-repository --repository-name "$IMAGE_NAME" --region "$AWS_REGION" 2>/dev/null || true

    echo "Pulling from GCR: $IMAGE"
    docker pull "$IMAGE"

    echo "Tagging for ECR: $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG"
    docker tag "$IMAGE" "$ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG"

    echo "Pushing to ECR..."
    aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ECR_REGISTRY"
    docker push "$ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG"
  fi
done

# Verification
echo "=== Verification ==="
echo "Listing ECR repositories..."
aws ecr describe-repositories --region "$AWS_REGION" --query 'repositories[].repositoryName' --output table
```

### 04-migrate-secrets.sh

Migrate secrets from GCP Secret Manager to AWS Secrets Manager:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Secrets migration: GCP Secret Manager → AWS Secrets Manager
# Usage: ./04-migrate-secrets.sh [--execute]

DRY_RUN=true
[[ "${1:-}" == "--execute" ]] && DRY_RUN=false

# TODO: List secrets to migrate
SECRETS=(
  "database-password"
  "api-key"
  # Add more secrets from your GCP project
)

for SECRET_NAME in "${SECRETS[@]}"; do
  if [ "$DRY_RUN" = true ]; then
    echo "[DRY RUN] Would migrate secret: $SECRET_NAME"
  else
    echo "Reading secret from GCP: $SECRET_NAME"
    SECRET_VALUE=$(gcloud secrets versions access latest --secret="$SECRET_NAME")

    echo "Creating secret in AWS: $SECRET_NAME"
    aws secretsmanager create-secret \
      --name "$SECRET_NAME" \
      --secret-string "$SECRET_VALUE" \
      --tags Key=MigrationSource,Value=gcp-secret-manager 2>/dev/null || \
    aws secretsmanager put-secret-value \
      --secret-id "$SECRET_NAME" \
      --secret-string "$SECRET_VALUE"
  fi
done

# Verification
echo "=== Verification ==="
aws secretsmanager list-secrets --query 'SecretList[].Name' --output table
```

### 05-validate-migration.sh

Post-migration validation script:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Post-migration validation
# Usage: ./05-validate-migration.sh

echo "=== Migration Validation ==="

# Check Terraform state
echo "--- Terraform Resources ---"
cd terraform/
terraform state list | wc -l
echo "resources in Terraform state"

# Check ECS services
echo "--- ECS Services ---"
aws ecs list-services --cluster "${PROJECT_NAME:-gcp-migration}" --query 'serviceArns' --output table 2>/dev/null || echo "No ECS cluster found"

# Check RDS instances
echo "--- RDS Instances ---"
aws rds describe-db-instances --query 'DBInstances[].{ID:DBInstanceIdentifier,Status:DBInstanceStatus,Endpoint:Endpoint.Address}' --output table 2>/dev/null || echo "No RDS instances found"

# Check S3 buckets
echo "--- S3 Buckets ---"
aws s3 ls | grep "${PROJECT_NAME:-gcp-migration}" || echo "No matching S3 buckets found"

# Check secrets
echo "--- Secrets Manager ---"
aws secretsmanager list-secrets --query 'SecretList[].Name' --output table 2>/dev/null || echo "No secrets found"

echo "=== Validation Complete ==="
echo "Review the output above. All resources should show healthy status."
echo "TODO: Run application-level health checks"
echo "TODO: Compare performance metrics against GCP baseline"
```

## Step 6: Self-Check

After generating all files, verify the following quality rules:

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

### Script Quality Rules

1. All scripts use `set -euo pipefail`
2. All scripts default to dry-run mode
3. All scripts include verification steps
4. All scripts are numbered for execution order
5. All TODO markers are clearly marked with context

## Phase Completion

Report the list of generated files to the parent orchestrator. **Do NOT update `.phase-status.json`** — the parent `generate.md` handles phase completion.

Output:

```
Generated infrastructure artifacts:
- terraform/main.tf
- terraform/variables.tf
- terraform/outputs.tf
- terraform/[domain].tf (for each domain with resources)
- scripts/01-validate-prerequisites.sh
- scripts/02-migrate-data.sh
- scripts/03-migrate-containers.sh
- scripts/04-migrate-secrets.sh
- scripts/05-validate-migration.sh

Total: [N] Terraform files, [N] migration scripts
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

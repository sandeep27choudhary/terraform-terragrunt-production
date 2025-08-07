# Terraform Project Analysis

## Table of Contents
1. [Project Overview](#project-overview)
2. [Directory Structure](#directory-structure)
3. [Terraform + Terragrunt Architecture](#terraform--terragrunt-architecture)
4. [Terragrunt Structure](#terragrunt-structure)
5. [GitHub Workflows](#github-workflows)
6. [Module Structure](#module-structure)
7. [Deployment Workflow](#deployment-workflow)
8. [Security and Compliance](#security-and-compliance)
9. [Best Practices](#best-practices)

## Project Overview

This repository manages infrastructure using Terraform and Terragrunt. The project follows a multi-brand, multi-environment approach with separate Terragrunt configurations for different brands and environments (stage, prod).

### Key Features
- **Multi-brand support**: Multiple brands with separate configurations
- **Multi-environment**: Stage and production environments
- **Modular architecture**: Reusable Terraform modules
- **Automated deployments**: GitHub Actions workflows
- **Security scanning**: Checkov integration
- **State management**: S3 backend with DynamoDB locking

## Directory Structure

```
terraform-project/
├── .github/                          # GitHub Actions workflows
│   ├── workflows/
│   │   ├── create-resource-branch-based-security.yml  # Main deployment workflow
│   │   ├── destroy-resource-on-aws.yml
│   │   ├── ensure-branch-merge.yml
│   │   ├── ensure-branch-prefix.yml
│   │   ├── created-pull-request.yml
│   │   └── auto-assign-reviewers.yml
│   ├── labeler.yml
│   └── reviewers.yml
├── modules/                          # Reusable Terraform modules
│   └── aws/
│       ├── alb/                      # Application Load Balancer
│       ├── cloudfront/               # CloudFront CDN
│       ├── documentdb/               # DocumentDB clusters
│       ├── dynamodb/                 # DynamoDB tables
│       ├── ecr/                      # Elastic Container Registry
│       ├── ecs/                      # ECS clusters and services
│       ├── eks/                      # EKS clusters
│       ├── lambdas/                  # Lambda functions
│       ├── monitoring/               # Monitoring and alerting
│       ├── rds/                   # RDS databases (v2)
│       └── vpc/                   # VPC and networking (v2)
├── terragrunt/                       # Terragrunt configurations
│   ├── brand1_terragrunt/            # Brand1 configuration
│   │   ├── brand1/
│   │   │   ├── prod/                 # Production environment
│   │   │   │   ├── vpc/              # VPC configuration
│   │   │   │   ├── ecs-cluster/      # ECS cluster
│   │   │   │   ├── ecr/              # ECR repositories
│   │   │   │   ├── prod-alb/         # Production ALB
│   │   │   │   └── ...               # Other resources
│   │   │   └── stage/                # Stage environment
│   │   ├── root-config.hcl           # Root configuration
│   │   └── terragrunt.hcl
│   └── brand2_terragrunt/            # Brand2 configuration
│       ├── brand2/
│       │   ├── prod/                 # Production environment
│       │   └── stage/                # Stage environment
│       ├── root-config.hcl           # Root configuration
│       └── terragrunt.hcl
├── generated-diagrams/               # Architecture diagrams
│   ├── brand1-prod-vpc-applications.png
│   └── brand1-prod-vpc-architecture.png
├── README.md
└── .gitignore
```

## Terraform + Terragrunt Architecture

### Terragrunt Configuration Structure

#### Root Configuration (`root-config.hcl`)
```hcl
locals {
  region = "us-east-1"  # Replace with your preferred region
  version_terraform    = ">1.8.0"
  version_terragrunt   = ">0.59.1"
  version_provider_aws = ">5.45.0"
  root_tags = {
    Brand = "brand1"  # Replace with your brand name
  }
}

terraform {
  after_hook "after_hook_plan" {
    commands = ["plan"]
    execute  = ["sh", "-c", "terraform show -json tfplan.binary | jq > ${get_parent_terragrunt_dir("root")}/plan.json"]
  }
}

remote_state {
  backend = "s3"
  config = {
    bucket         = "brand1-tf-state-s3"  # Replace with your bucket name
    key            = "${path_relative_to_include()}/terraform.tfstate"
    encrypt        = true
    region         = local.region
    dynamodb_table = "brand1-tf-state-dynamo-01"  # Replace with your table name
  }
}
```

#### Environment Configuration (`prod.hcl` / `stage.hcl`)
```hcl
locals {
  environment = "prod"  # or "stage"
  brand = "brand1"      # Replace with your brand name
  
  tags = {
    environment = local.environment
    developer   = "developer-name"  # Replace with developer name
    brand       = local.brand
  }
}
```

#### Resource Configuration (`terragrunt.hcl`)
```hcl
include "root" {
  path   = find_in_parent_folders("root-config.hcl")
  expose = true
}

include "stage" {
  path   = find_in_parent_folders("prod.hcl")
  expose = true
}

locals {
  local_tags = {}
  tags = merge(include.root.locals.root_tags, include.stage.locals.tags, local.local_tags)
}

generate "provider_global" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<EOF
terraform {
  backend "s3" {}
  required_version = "${include.root.locals.version_terraform}"
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "${include.root.locals.version_provider_aws}"
    }
  }
}

provider "aws" {
  region = "${include.root.locals.region}"
}
EOF
}

inputs = {
  vpc_subnet_module = {
    name                 = "brand1-prod-vpc"  # Replace with your VPC name
    cidr_block           = "10.22.0.0/16"     # Replace with your CIDR block
    azs                  = ["us-east-1a", "us-east-1b", "us-east-1c"]  # Replace with your AZs
    private_subnets      = ["10.22.3.0/24", "10.22.4.0/24", "10.22.5.0/24"]  # Replace with your private subnets
    public_subnets       = ["10.22.0.0/24", "10.22.1.0/24", "10.22.2.0/24"]   # Replace with your public subnets
    enable_ipv6          = false
    enable_nat_gateway   = true
    single_nat_gateway   = false
    one_nat_gateway_per_az = false
    enable_vpn_gateway   = false
    enable_dns_hostnames = true
    enable_dns_support   = true
    nat_gateway_azs      = ["us-east-1a", "us-east-1b"]  # Replace with your NAT gateway AZs
  }
  tags = local.tags
}

terraform {
  source = "${get_parent_terragrunt_dir("root")}../../..//modules/aws/vpc_v2"
}
```

## Terragrunt Structure

### Overview

The Terragrunt structure follows a hierarchical approach with clear separation of concerns:
- **Brand-level separation**: Each brand has its own Terragrunt configuration
- **Environment-level separation**: Each brand has stage and production environments
- **Resource-level separation**: Each environment contains specific AWS resources
- **Dependency management**: Resources can reference outputs from other resources

### Detailed Terragrunt Directory Structure

```
terragrunt/
├── brand1_terragrunt/                      # Brand1 configuration
│   ├── root-config.hcl                     # Root configuration for Brand1
│   ├── terragrunt.hcl                      # Root terragrunt configuration
│   └── brand1/                             # Brand1 resources
│       ├── prod/                           # Production environment
│       │   ├── prod.hcl                    # Production environment configuration
│       │   ├── vpc/                        # VPC and networking
│       │   │   ├── terragrunt.hcl          # VPC configuration
│       │   │   └── .terragrunt-cache/      # Terragrunt cache
│       │   ├── security_groups/            # Security groups
│       │   │   ├── terragrunt.hcl          # Security groups configuration
│       │   │   └── .terragrunt-cache/
│       │   ├── ecs-cluster/                # ECS cluster
│       │   │   ├── terragrunt.hcl          # ECS cluster configuration
│       │   │   └── .terragrunt-cache/
│       │   ├── ecr/                        # ECR repositories
│       │   │   ├── terragrunt.hcl          # ECR configuration
│       │   │   └── .terragrunt-cache/
│       │   ├── prod-alb/                   # Production ALB
│       │   │   ├── terragrunt.hcl          # ALB configuration
│       │   │   └── .terragrunt-cache/
│       └── stage/                          # Stage environment
│           ├── stage.hcl                   # Stage environment configuration
│           ├── vpc/                        # VPC and networking
│           ├── security_groups/            # Security groups
└── brand2_terragrunt/                      # Brand2 configuration
    ├── root-config.hcl                     # Root configuration for Brand2
    ├── terragrunt.hcl                      # Root terragrunt configuration
    └── brand2/                             # Brand2 resources
        ├── prod/                           # Production environment
        │   ├── prod.hcl                    # Production environment configuration
        │   ├── vpc/                        # VPC and networking
        └── stage/                          # Stage environment
            ├── stage.hcl                   # Stage environment configuration
            ├── vpc/                        # VPC and networking
```

### Terragrunt Configuration Hierarchy

#### 1. Root Configuration (`root-config.hcl`)

**Purpose**: Defines global settings for the entire brand
**Location**: `terragrunt/{brand}_terragrunt/root-config.hcl`

```hcl
locals {
  region = "us-east-1"  # Replace with your preferred region
  version_terraform    = ">1.8.0"
  version_terragrunt   = ">0.59.1"
  version_provider_aws = ">5.45.0"
  root_tags = {
    Brand = "brand1"  # Replace with your brand name
  }
}

terraform {
  after_hook "after_hook_plan" {
    commands = ["plan"]
    execute  = ["sh", "-c", "terraform show -json tfplan.binary | jq > ${get_parent_terragrunt_dir("root")}/plan.json"]
  }
}

remote_state {
  backend = "s3"
  config = {
    bucket         = "brand1-tf-state-s3"  # Replace with your bucket name
    key            = "${path_relative_to_include()}/terraform.tfstate"
    encrypt        = true
    region         = local.region
    dynamodb_table = "brand1-tf-state-dynamo-01"  # Replace with your table name
  }
}
```

#### 2. Environment Configuration (`prod.hcl` / `stage.hcl`)

**Purpose**: Defines environment-specific settings
**Location**: `terragrunt/{brand}_terragrunt/{brand}/{environment}/`

```hcl
locals {
  environment = "prod"  # or "stage"
  brand = "brand1"      # Replace with your brand name
  
  tags = {
    environment = local.environment
    developer   = "developer-name"  # Replace with developer name
    brand       = local.brand
  }
}
```

#### 3. Resource Configuration (`terragrunt.hcl`)

**Purpose**: Defines specific resource configurations
**Location**: `terragrunt/{brand}_terragrunt/{brand}/{environment}/{resource}/`

**Example - VPC Configuration**:
```hcl
include "root" {
  path   = find_in_parent_folders("root-config.hcl")
  expose = true
}

include "stage" {
  path   = find_in_parent_folders("prod.hcl")
  expose = true
}

locals {
  local_tags = {}
  tags = merge(include.root.locals.root_tags, include.stage.locals.tags, local.local_tags)
}

generate "provider_global" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<EOF
terraform {
  backend "s3" {}
  required_version = "${include.root.locals.version_terraform}"
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "${include.root.locals.version_provider_aws}"
    }
  }
}

provider "aws" {
  region = "${include.root.locals.region}"
}
EOF
}

inputs = {
  vpc_subnet_module = {
    name                 = "brand1-prod-vpc"  # Replace with your VPC name
    cidr_block           = "10.22.0.0/16"     # Replace with your CIDR block
    azs                  = ["us-east-1a", "us-east-1b", "us-east-1c"]  # Replace with your AZs
    private_subnets      = ["10.22.3.0/24", "10.22.4.0/24", "10.22.5.0/24"]  # Replace with your private subnets
    public_subnets       = ["10.22.0.0/24", "10.22.1.0/24", "10.22.2.0/24"]   # Replace with your public subnets
    enable_ipv6          = false
    enable_nat_gateway   = true
    single_nat_gateway   = false
    one_nat_gateway_per_az = false
    enable_vpn_gateway   = false
    enable_dns_hostnames = true
    enable_dns_support   = true
    nat_gateway_azs      = ["us-east-1a", "us-east-1b"]  # Replace with your NAT gateway AZs
  }
  tags = local.tags
}

terraform {
  source = "${get_parent_terragrunt_dir("root")}../../..//modules/aws/vpc_v2"
}
```

**Example - ECS Cluster Configuration (with dependencies)**:
```hcl
include "root" {
  path   = find_in_parent_folders("root-config.hcl")
  expose = true
}

include "stage" {
  path   = find_in_parent_folders("prod.hcl")
  expose = true
}

locals {
  local_tags = {
    service = "ecs-cluster"
    purpose = "container-orchestration"
  }
  
  stage_tags_clean = {
    environment = include.stage.locals.tags.environment
    developer   = include.stage.locals.tags.developer
  }
  
  tags = merge(include.root.locals.root_tags, local.stage_tags_clean, local.local_tags)
}

# Dependencies on VPC for networking context
dependency "vpc" {
  config_path = "../vpc"
  
  mock_outputs = {
    vpc_id               = "vpc-mock"
    vpc_cidr_block       = "10.0.0.0/16"
    private_subnet_ids   = ["subnet-mock-1", "subnet-mock-2", "subnet-mock-3"]
    public_subnet_ids    = ["subnet-mock-1", "subnet-mock-2", "subnet-mock-3"]
  }
}

# Dependencies on security groups
dependency "security_groups" {
  config_path = "../security_groups"
  
  mock_outputs = {
    security_group_ids = {
      brand1_stage_SG-ECS = "sg-mock-ecs"
      brand1_stage_SG-Private-Access = "sg-mock-private"
    }
  }
}

inputs = {
  ecs_cluster_name = "prod-brand1-cluster"  # Replace with your cluster name
  environment      = include.stage.locals.environment
  organization     = include.stage.locals.brand
  service          = "ecs-cluster"
  
  enable_container_insights = true
  kms_deletion_window      = 7
  log_retention_days       = 30
  
  enable_fargate           = true
  enable_fargate_spot      = true
  
  tags = local.tags
}

terraform {
  source = "${get_parent_terragrunt_dir("root")}../../..//modules/aws/ecs"
}
```

### Key Terragrunt Features Used

#### 1. Dependency Management
- **`dependency` blocks**: Define dependencies between resources
- **`mock_outputs`**: Provide mock outputs for planning when dependencies don't exist
- **`config_path`**: Reference other Terragrunt configurations

#### 2. Configuration Inheritance
- **`include` blocks**: Inherit configurations from parent directories
- **`find_in_parent_folders`**: Find configuration files in parent directories
- **`expose = true`**: Expose local variables to child configurations

#### 3. Dynamic Configuration
- **`generate` blocks**: Generate Terraform files dynamically
- **`inputs`**: Pass variables to Terraform modules
- **`locals`**: Define local variables and merge configurations

#### 4. State Management
- **S3 backend**: Centralized state storage
- **DynamoDB locking**: Prevent concurrent modifications
- **Encryption**: State files encrypted at rest
- **Versioning**: S3 versioning for state files

### Resource Organization Strategy

#### 1. Foundational Resources
- **VPC**: Network infrastructure (first to deploy)
- **Security Groups**: Network security (depends on VPC)
- **IAM Roles**: Access control (can be deployed early)

#### 2. Core Infrastructure
- **ECS Cluster**: Container orchestration (depends on VPC, security groups)
- **ECR**: Container registry (depends on IAM)
- **ALB**: Load balancing (depends on VPC, security groups)

#### 3. Application Resources
- **RDS**: Databases (depends on VPC, security groups)
- **S3**: Storage (depends on IAM)
- **ECS Services**: Applications (depends on ECS cluster, ALB)

#### 4. Monitoring and Observability
- **CloudWatch**: Logging and monitoring
- **SNS**: Notifications
- **Sentry**: Error tracking

### Best Practices Implemented

1. **Separation of Concerns**: Clear separation between brands, environments, and resources
2. **Dependency Management**: Proper dependency declaration with mock outputs
3. **Configuration Reuse**: DRY principle through configuration inheritance
4. **State Organization**: Logical state file organization by brand/environment/resource
5. **Security**: Encrypted state storage and IAM-based access control
6. **Versioning**: Consistent version management across environments
7. **Tagging**: Comprehensive tagging strategy for resource management
8. **Documentation**: Clear configuration structure and comments

## GitHub Workflows

### 1. Main Deployment Workflow (`create-resource-branch-based-security.yml`)

**Purpose**: Deploy infrastructure resources with branch-based security controls

**Key Features**:
- **Branch-based security**: Only `main` branch can deploy to prod, `develop` and `feature/*` branches can deploy to stage
- **Multi-brand support**: Supports multiple brands with separate configurations
- **Security scanning**: Integrates Checkov for security compliance
- **Slack notifications**: Sends deployment reports to Slack
- **Plan generation**: Creates JSON plan files for review

**Workflow Steps**:
1. **Environment Setup**: Sets resource environment based on brand selection
2. **Branch Validation**: Enforces branch rules for different environments
3. **AWS Configuration**: Configures AWS credentials based on brand
4. **Security Scanning**: Runs Checkov on Terraform plan
5. **Deployment**: Executes Terraform plan/apply
6. **Notification**: Sends Slack notification with results

### 2. Lambda Deployment Workflow (`deploy-lambdas.yml`)

**Purpose**: Build and deploy Lambda functions

**Key Features**:
- **Python support**: Sets up Python 3.12 environment
- **Dependency management**: Installs requirements.txt dependencies
- **Artifact creation**: Creates ZIP files for Lambda deployment
- **Multi-environment**: Supports dev, stage, pre-prod, prod environments

### 3. Other Workflows

- **`destroy-resource-on-aws.yml`**: Safely destroy AWS resources
- **`ensure-branch-merge.yml`**: Ensures proper branch merging
- **`ensure-branch-prefix.yml`**: Enforces branch naming conventions
- **`auto-assign-reviewers.yml`**: Automatically assigns reviewers to PRs

## Module Structure

### Module Organization

Each module follows a consistent structure:
```
module_name/
├── main.tf           # Main resource definitions
├── variables.tf      # Input variables
├── outputs.tf        # Output values
├── data-source.tf    # Data sources (if needed)
└── README.md         # Module documentation
```

### Key Modules

#### 1. VPC Module (`vpc_v2/`)
- **Purpose**: Creates VPC with public/private subnets, NAT gateways, and routing
- **Features**: Multi-AZ support, IPv6 support, NAT gateway configuration
- **Dependencies**: None (foundational module)

#### 2. ECS Module (`ecs_v3/`)
- **Purpose**: Deploys ECS clusters and services
- **Features**: Fargate support, service discovery, load balancer integration
- **Dependencies**: VPC, security groups, ALB

#### 3. RDS Module (`rds_v3/`)
- **Purpose**: Deploys RDS databases
- **Features**: Multi-AZ support, encryption, backup configuration
- **Dependencies**: VPC, security groups

#### 4. Security Groups (`security_group_v2/`)
- **Purpose**: Creates security groups with ingress/egress rules
- **Features**: Flexible rule configuration, tag-based organization
- **Dependencies**: VPC

## Deployment Workflow

### 1. Local Development
```bash
# Navigate to specific resource
cd terragrunt/brand1_terragrunt/brand1/prod/vpc

# Initialize Terragrunt
terragrunt init

# Plan changes
terragrunt plan

# Apply changes
terragrunt apply
```

### 2. GitHub Actions Deployment

1. **Manual Trigger**: Use GitHub Actions UI to trigger deployment
2. **Brand Selection**: Choose brand (Brand1, Brand2, etc.)
3. **Environment Selection**: Choose environment (stage, prod)
4. **Resource Selection**: Choose resource to deploy
5. **Operation Selection**: Choose plan or apply
6. **Execution**: Workflow executes with security checks

### 3. Branch Protection Rules

- **Production**: Only `main` branch can apply changes
- **Stage**: `develop` and `feature/*` branches can apply changes
- **Development**: All branches can plan changes

## Security and Compliance

### 1. Checkov Integration
- **Purpose**: Security and compliance scanning
- **Integration**: Runs on every Terraform plan
- **Output**: Generates security reports
- **Configuration**: Uses `--framework terraform_plan` for plan-based scanning

### 2. State Management
- **Backend**: S3 with encryption enabled
- **Locking**: DynamoDB for state locking
- **Versioning**: S3 versioning for state files
- **Access Control**: IAM-based access control

### 3. Secrets Management
- **AWS Credentials**: Stored as GitHub Secrets
- **Brand-specific**: Separate credentials for each brand
- **Environment-specific**: Different credentials for different environments

## Best Practices

### 1. Module Design
- **Reusability**: Modules are designed to be reusable across brands
- **Versioning**: Modules use semantic versioning
- **Documentation**: Each module has README documentation
- **Testing**: Modules include example configurations

### 2. Terragrunt Configuration
- **DRY Principle**: Uses Terragrunt's DRY features to avoid repetition
- **Environment Separation**: Clear separation between environments
- **Tag Management**: Consistent tagging strategy
- **State Organization**: Logical state file organization

### 3. Security
- **Least Privilege**: IAM roles follow least privilege principle
- **Encryption**: All data encrypted at rest and in transit
- **Network Security**: Security groups restrict access
- **Compliance**: Regular security scanning with Checkov

### 4. Monitoring and Alerting
- **CloudWatch**: Comprehensive logging and monitoring
- **SNS**: Notification system for alerts
- **Slack Integration**: Real-time deployment notifications
- **Health Checks**: Automated health check monitoring

## Conclusion

This Terraform project demonstrates a well-architected, enterprise-grade infrastructure management solution. Key strengths include:

1. **Scalability**: Multi-brand, multi-environment support
2. **Security**: Comprehensive security controls and compliance
3. **Automation**: Fully automated deployment pipelines
4. **Maintainability**: Modular design with clear separation of concerns
5. **Monitoring**: Integrated monitoring and alerting
6. **Compliance**: Security scanning and audit trails

The project follows industry best practices and provides a solid foundation for managing complex, multi-tenant AWS infrastructure. 

# AWS MVP File Storage Service

This repository contains Terraform infrastructure as code (IaC) for deploying an AWS-based MVP file storage service with a Flask web application.

## 📦 Project Deliverables

This project fulfills the AWS MVP requirements with the following deliverables:

### ✅ Architecture Diagram
- **File**: `mvp.drawio.png`
- **Features**: 
  - Visual representation of all AWS components
  - Network topology with VPC, subnets, and security groups
  - Module references and configuration details
  - Data flow connections
  - Monitoring and state management

### ✅ Terraform Code
- **Main Configuration**: `main.tf`, `variables.tf`, `outputs.tf`
- **Modular Structure**:
  - `modules/vpc/` - Network infrastructure (using terraform-aws-modules/vpc/aws)
  - `modules/ec2/` - Compute resources with security groups
  - `modules/rds/` - Database resources with security groups
  - `modules/s3/` - Storage resources with encryption and lifecycle policies
- **Features**:
  - Comprehensive comments explaining each resource
  - Input variables for customization
  - Output values for easy access to resource information
  - Remote state management with S3 and DynamoDB

### ✅ Documentation
- **README.md**: Complete project documentation with deployment instructions
- **BEST_PRACTICES.md**: Modularization, naming conventions, security, and state management
- **DESIGN_DECISIONS.md**: Architectural choices and trade-offs explained
- **VALIDATION_GUIDE.md**: How to validate and format Terraform code

### ✅ MVP Requirements Met
- [x] Define MVP Requirements (microservice with database and storage)
- [x] Design Architecture Diagram (showing all components and connections)
- [x] Terraform Modeling (modular structure with network, compute, database modules)
- [x] Best Practices Checklist (modularization, naming, state management, security)
- [x] Documentation & Deliverables (architecture diagram, Terraform code, comprehensive README)

## Architecture Overview

![AWS MVP File Storage Service Architecture](mvp.drawio.png)

The architecture diagram above illustrates the complete infrastructure setup with Terraform module references and configuration details.

### Core Components

- **End Users**: External users accessing the application via HTTPS/HTTP (ports 80/443)
- **Internet Gateway**: Entry point for internet traffic into the VPC
- **IAM Role**: Grants EC2 instances permissions to access S3 buckets
- **VPC (10.0.0.0/16)**: Virtual Private Cloud in us-west-2 region
  - Module: `terraform-aws-modules/vpc/aws` (v6.0.0)
  - **Public Subnets** (10.0.1.0/24, 10.0.2.0/24): Host the EC2 Flask web application
  - **Private Subnets** (10.0.11.0/24, 10.0.12.0/24): Host the RDS PostgreSQL database
- **EC2 Instance**: Runs the Flask web application
  - Module: `./modules/ec2`
  - AMI: Ubuntu 24.04 (ami-061b09f4833e8c74a)
  - Instance Type: t3.micro
  - Security Group: SSH (22), HTTP (80), HTTPS (443)
  - EBS Volume: Encrypted
- **RDS PostgreSQL Database**: Managed database service
  - Module: `./modules/rds`
  - Engine: PostgreSQL 12.7
  - Instance Class: db.t3.micro
  - Storage: 20GB (encrypted at rest)
  - Security Group: Only accepts connections from EC2 on port 5432
- **S3 Bucket**: Encrypted file storage with lifecycle management
  - Module: `./modules/s3`
  - Naming: `mvp-app-storage-{env}`
  - Encryption: AES256 server-side encryption
  - Versioning: Enabled
  - Lifecycle Policy: Move to Glacier after 30 days, delete after 365 days
- **Terraform State Backend**: S3 bucket with DynamoDB locking
  - Bucket: `butterfly521113-tf-state`
  - Key: `mvp/state`
  - Region: us-west-2
  - DynamoDB Table: `terraform-state-lock`
- **CloudWatch**: Centralized monitoring, metrics collection, and access logs

### Security Features

- **Security Groups**: Control inbound/outbound traffic
  - EC2 allows SSH (22), HTTP (80), and HTTPS (443) from internet
  - RDS only accepts connections from EC2 security group on port 5432
- **Encryption**: All data encrypted at rest
  - S3: AES256 server-side encryption
  - EBS: Encrypted volumes
  - RDS: Storage encryption enabled
- **Private Networking**: Database isolated in private subnets with no direct internet access
- **IAM Roles**: Least privilege access control for EC2 to S3 communication
- **State Management**: Terraform state stored securely in S3 with DynamoDB locking

### Data Flow

1. **User Request**: Users connect via HTTPS/HTTP through the Internet Gateway
2. **Web Application**: EC2 instance serves the Flask web application in public subnet
3. **Database Operations**: Application queries the PostgreSQL database in private subnet (port 5432)
4. **File Storage**: Application uploads/downloads files to/from S3 bucket using IAM role
5. **Monitoring**: All components send metrics and logs to CloudWatch for observability

### Terraform Configuration

The infrastructure is organized into reusable Terraform modules:

```
.
├── main.tf                 # Root configuration
├── variables.tf            # Input variables
├── outputs.tf              # Output values
└── modules/
    ├── vpc/                # VPC module (using community module)
    │   ├── main.tf         # VPC instance and security group
    │   ├── variables.tf    # Module variables
    │   └── outputs.tf      # Module outputs
    ├── ec2/                # EC2 web server module
    │   ├── main.tf         # EC2 instance and security group
    │   ├── variables.tf    # Module variables
    │   └── outputs.tf      # Module outputs
    ├── rds/                # RDS database module
    │   ├── main.tf         # RDS instance and security group
    │   ├── variables.tf    # Module variables
    │   └── outputs.tf      # Module outputs
    └── s3/                 # S3 storage module
        ├── main.tf         # S3 bucket with encryption and lifecycle
        ├── variables.tf    # Module variables
        └── outputs.tf      # Module outputs
```

### Key Configuration Details

**Network Module** (`modules/vpc/main.tf`):
- Uses official AWS VPC module (v6.0.0)
- CIDR: 10.0.0.0/16
- NAT Gateway: Disabled (cost optimization for MVP)

**Compute Module** (`modules/ec2/main.tf`):
- Security group with controlled ingress/egress
- Encrypted root block device
- Deployed in public subnet for direct internet access

**Database Module** (`modules/rds/main.tf`):
- PostgreSQL 12.7 engine
- Multi-AZ: Disabled (MVP configuration)
- Storage encryption enabled
- Skip final snapshot for easier teardown

**Storage Module** (`modules/s3/main.tf`):
- Server-side encryption with AES256
- Versioning for data protection
- Lifecycle rules for cost optimization
- Public access blocked by default

## Deployment

### Prerequisites

- Terraform >= 1.0
- AWS CLI configured with appropriate credentials
- S3 bucket for Terraform state (`butterfly521113-tf-state`)
- DynamoDB table for state locking (`terraform-state-lock`)

### Deployment Steps

1. **Configure Variables**: Copy and customize the example configuration
   ```bash
   cp terraform.tfvars.example terraform.tfvars
   # Edit terraform.tfvars with your specific values
   ```
   
   Or create a `terraform.tfvars` file manually:
   ```hcl
   region            = "us-west-2"
   env               = "dev"
   instance_type     = "t3.micro"
   key_name          = "your-ssh-key"
   db_username       = "admin"
   db_password       = "Admin123456!"
   db_instance_class = "db.t3.micro"
   db_allocated_storage = 20
   ```

2. **Format and Validate** (see [VALIDATION_GUIDE.md](VALIDATION_GUIDE.md) for details):
   ```bash
   # Format Terraform files
   terraform fmt -recursive
   
   # Initialize Terraform
   terraform init
   
   # Validate configuration
   terraform validate
   ```

3. **Review Plan**:
   ```bash
   terraform plan
   ```

4. **Apply Configuration**:
   ```bash
   terraform apply
   ```

5. **Access Outputs**:
   ```bash
   terraform output ec2_public_ip
   terraform output rds_endpoint
   terraform output s3_bucket
   ```

### Teardown

To destroy all resources:
```bash
terraform destroy
```

## Outputs

After deployment, Terraform provides these outputs:

- `ec2_public_ip`: Public IP address of the Flask web server
- `ec2_public_dns`: Public DNS name for the EC2 instance
- `sg_id`: Security group ID for the EC2 instance
- `rds_endpoint`: RDS database connection endpoint
- `db_username`: RDS database username (sensitive)
- `s3_bucket`: S3 bucket name for file storage
- `vpc_id`: VPC identifier
- `public_subnets`: List of public subnet IDs
- `private_subnets`: List of private subnet IDs

## Cost Optimization

This MVP configuration is optimized for cost:
- **EC2**: t3.micro instance (free tier eligible)
- **RDS**: db.t3.micro instance (free tier eligible)
- **S3**: Pay-per-use with lifecycle policies to Glacier
- **VPC**: No NAT Gateway (cost saving)
- **Multi-AZ**: Disabled for database (development mode)

## Security Best Practices

- All storage encrypted at rest
- Database in private subnet with no internet access
- Security groups follow least privilege principle
- IAM roles instead of access keys for EC2-S3 communication
- Terraform state stored in encrypted S3 bucket
- State locking prevents concurrent modifications

## Next Steps

To enhance this MVP for production:
1. Enable NAT Gateway for private subnet internet access
2. Enable RDS Multi-AZ for high availability
3. Add Application Load Balancer for scalability
4. Implement Auto Scaling for EC2 instances
5. Add AWS WAF for web application firewall
6. Restrict SSH access to specific IP addresses
7. Implement CloudWatch alarms and notifications
8. Add AWS Backup for automated backups

# Terraform for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**Providers, Resources, Variables, and Outputs**
```hcl
# Provider configuration (AWS example)
provider "aws" {
  region = var.aws_region
}

# Variable declaration
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

# Resource definition
resource "aws_s3_bucket" "my_bucket" {
  bucket = "company-data-bucket-${var.environment}"
  
  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Output value
output "bucket_name" {
  description = "The name of the created S3 bucket"
  value       = aws_s3_bucket.my_bucket.id
}
```
* **provider**: Tells Terraform which cloud/API to interact with.
* **resource**: Defines a specific infrastructure object (e.g., EC2, S3, Azure VM).
* **variable**: Input parameters to make configurations reusable.
* **output**: Extracts useful information from the state after deployment.

---

## 2. Intermediate Examples

**Data Sources, Locals, and Remote State (Azure/AWS)**
```hcl
# Remote Backend (S3 with DynamoDB for state locking)
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}

# Local Values (for complex expressions or repeated values)
locals {
  common_tags = {
    Project     = "PaymentGateway"
    Owner       = "DevOpsTeam"
    Environment = terraform.workspace
  }
  name_prefix = "pg-${terraform.workspace}"
}

# Data Source (Fetch existing infrastructure data)
data "aws_ami" "latest_amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "app_server" {
  ami           = data.aws_ami.latest_amazon_linux.id
  instance_type = "t3.micro"
  tags          = merge(local.common_tags, { Name = "${local.name_prefix}-app" })
}
```

---

## 3. Advanced Examples

**Modules, `for_each`, Dynamic Blocks, and Workspaces**
```hcl
# 1. Modules (Reusing code)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${local.name_prefix}-vpc"
  cidr = var.vpc_cidr
  
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  
  enable_nat_gateway = true
}

# 2. for_each (Creating multiple resources from a map or set)
variable "users" {
  type = map(object({
    department = string
    role       = string
  }))
  default = {
    "alice" = { department = "engineering", role = "developer" }
    "bob"   = { department = "finance", role = "auditor" }
  }
}

resource "aws_iam_user" "team" {
  for_each = var.users
  name     = each.key
  tags = {
    Department = each.value.department
    Role       = each.value.role
  }
}

# 3. Dynamic Blocks (Iterating inside a resource block)
variable "ingress_ports" {
  type    = list(number)
  default = [80, 443, 8080]
}

resource "aws_security_group" "web_sg" {
  name   = "web-ports-sg"
  vpc_id = module.vpc.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

---

## 4. Interview Coding Exercises

### Problem 1: Conditional Resource Creation
**Task:** Write a Terraform snippet that creates an Azure Storage Account. Only create it if the variable `create_storage` is set to `true`.

**Solution:**
```hcl
variable "create_storage" {
  type    = bool
  default = false
}

resource "azurerm_storage_account" "example" {
  count                    = var.create_storage ? 1 : 0
  name                     = "storageacct${count.index}"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```
**Explanation:** Using the `count` meta-argument with a ternary operator (`condition ? true_val : false_val`) is the standard way to conditionally create resources in Terraform.

### Problem 2: Transforming Data with `for_each`
**Task:** You have a list of subnet CIDR blocks: `["10.0.1.0/24", "10.0.2.0/24"]`. Create an AWS subnet for each using `for_each` (not `count`).

**Solution:**
```hcl
variable "subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

resource "aws_subnet" "main" {
  # for_each requires a map or a set of strings. We convert the list to a set.
  for_each   = toset(var.subnet_cidrs)
  vpc_id     = aws_vpc.main.id
  cidr_block = each.value
}
```
**Explanation:** `for_each` cannot iterate directly over a list if the list items could change order (which would mess up state). Converting to a `set` using `toset()` guarantees uniqueness and stability.

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Dependency Issue
**What is wrong here?**
```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
}

resource "aws_eip" "ip" {
  instance = aws_instance.web.id
}

output "web_public_ip" {
  value = aws_eip.ip.public_ip
}
```
**Answer:** EIPs require an Internet Gateway to exist in the VPC before they can be attached to an instance. If the VPC is custom and the IGW is created in the same apply, Terraform won't know the EIP depends on the IGW unless specified.
**Corrected:**
```hcl
resource "aws_eip" "ip" {
  instance   = aws_instance.web.id
  depends_on = [aws_internet_gateway.gw] # Explicit dependency
}
```

### Broken Configuration 2: State Lock Error
**Scenario:** You run `terraform apply` and get: `Error acquiring the state lock: ConditionalCheckFailedException`.
**What is wrong?** Another process (or teammate, or CI/CD pipeline) is currently running `apply` or the previous run crashed without releasing the lock.
**Solution:** Wait for the other process. If you are 100% sure no one is running it (crashed state), run `terraform force-unlock <LOCK_ID>`.

---

## 6. Common Interview Questions

**Q: "Write a Terraform module structure from scratch for a scalable web tier."**
*Answer Checklist:*
- `main.tf` (Resources: Auto Scaling Group, Load Balancer)
- `variables.tf` (Instance type, min/max instances, VPC IDs)
- `outputs.tf` (Load Balancer DNS name)
- `providers.tf` (Provider requirements)
- Mention how you'd call this module in a parent `main.tf`.

**Q: "What happens if you delete a resource manually in the AWS console that Terraform manages?"**
*Answer:* The next time you run `terraform plan`, Terraform will compare the state file with the real world (via API calls). It will notice the resource is missing in the real world and will output a plan to *recreate* it.

**Q: "How do you manage secrets in Terraform?"**
*Answer:* 
1. Never hardcode them. 
2. Pass them at runtime using environment variables (`TF_VAR_db_password`).
3. Fetch them dynamically using data sources from AWS Secrets Manager or HashiCorp Vault.
4. Keep the state file secure (remote backend with encryption and strict IAM policies), because secrets passed to resources will be saved in plaintext in the `.tfstate` file.

---

## 7. Cheat Sheet

| Command | Usage (4+ Yrs Experience Focus) |
| :--- | :--- |
| `terraform init -reconfigure` | Run when changing backend config to ignore existing state data. |
| `terraform plan -out=tfplan` | Save plan for CI/CD to ensure `apply` executes exact same plan. |
| `terraform apply tfplan` | Apply the saved plan. |
| `terraform import aws_iam_user.u bob` | Bring manually created resource "bob" into TF state. |
| `terraform state rm <resource>` | Remove resource from state (stops managing it) without destroying it. |
| `terraform state mv <old> <new>` | Rename a resource in state without destroying/recreating it (crucial for refactoring). |
| `terraform taint <resource>` | (Deprecated in v0.15.2+) Use `terraform apply -replace="<resource>"` instead to force recreation. |
| `terraform workspace new dev` | Create a new workspace (creates isolated state). |

# Cloud (AWS & Azure) for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax / CLI Commands

**Compute and Storage**

*AWS CLI*
```bash
# List all running EC2 instances (filtered)
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query "Reservations[*].Instances[*].InstanceId" --output text

# Copy a local file to S3
aws s3 cp backup.tar.gz s3://my-company-backups/database/
```

*Azure CLI*
```bash
# List all VMs in a resource group
az vm list --resource-group MyResourceGroup --query "[].name" --output tsv

# Upload a file to Blob Storage
az storage blob upload --account-name mystorageacct --container-name backups --file backup.tar.gz --name database/backup.tar.gz
```

---

## 2. Intermediate Examples

**Networking (VPCs / VNets) and Serverless**

*AWS VPC Architecture Concepts*
- **VPC:** The virtual network (e.g., `10.0.0.0/16`).
- **Public Subnet:** Has a route to an **Internet Gateway (IGW)**. Resources here (like Load Balancers) get public IPs.
- **Private Subnet:** No route to IGW. Uses a **NAT Gateway** to reach the internet (to download patches). Resources here (DBs, App Servers) have no public IPs.

*Azure VNet Architecture Concepts*
- **VNet:** The virtual network.
- **Subnets:** Segments of the VNet.
- **NSG (Network Security Group):** Acts as a firewall for subnets or individual NICs.
- Azure allows subnets to reach the internet by default unless blocked by NSG or routed through an Azure Firewall / NAT Gateway.

*Serverless (Lambda / Azure Functions)*
- Usually deployed via frameworks like Serverless Framework, AWS SAM, Terraform, or Azure DevOps.
- Key configurations: Memory size, Timeout, IAM Execution Role / Managed Identity (what other services the function is allowed to talk to).

---

## 3. Advanced Examples

**IAM, AssumeRole, and Managed Identities**

*AWS AssumeRole (Cross-Account Access)*
In DevOps, you rarely use static access keys. Instead, a CI/CD pipeline in Account A "assumes a role" in Account B to deploy resources.
```bash
# CLI command to assume a role
aws sts assume-role --role-arn arn:aws:iam::123456789012:role/DeployRole --role-session-name CIDeploy
# Returns temporary AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, and AWS_SESSION_TOKEN
```

*Azure Managed Identities*
Instead of storing Azure credentials inside a VM or Azure App Service, you enable a "Managed Identity". Azure automatically creates an identity in Azure AD. You grant that identity access to Key Vault or SQL. The application requests a token from a local metadata endpoint (no passwords required).

---

## 4. Interview Coding Exercises

### Problem 1: S3 Bucket Policy
**Task:** Write an AWS S3 Bucket Policy (JSON) that forces all uploads to the bucket `my-secure-bucket` to be encrypted with AES256.

**Solution:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireEncryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-secure-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

### Problem 2: Find Untagged Resources (Azure)
**Task:** Write a bash script using Azure CLI to find all Resource Groups that do NOT have an 'Environment' tag.
**Solution:**
```bash
#!/bin/bash
az group list --query "[?tags.Environment == null].name" -o tsv
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: AWS EC2 Cannot Reach Internet
**Scenario:** You deployed an EC2 instance in a private subnet. It needs to download packages from Ubuntu servers, but the connection times out.
**Troubleshooting Steps (The Interview Answer):**
1. **Security Group:** Does the instance SG allow outbound traffic (Outbound rules generally default to Allow All, but check it)?
2. **NACL:** Are Network ACLs blocking outbound traffic on port 80/443 or inbound ephemeral ports?
3. **Route Table:** Does the private subnet's Route Table have a route (`0.0.0.0/0`) pointing to a **NAT Gateway**?
4. **NAT Gateway:** Is the NAT Gateway deployed in a *Public* Subnet? Does that Public Subnet have a route to an Internet Gateway (IGW)? Does the NAT Gateway have an Elastic IP attached?

### Broken Configuration 2: Azure VM RDP Fails
**Scenario:** You cannot RDP (port 3389) into an Azure VM with a public IP.
**Troubleshooting Steps:**
1. Check the **NSG (Network Security Group)** attached to the VM's Network Interface. Is there an inbound rule allowing port 3389 from your specific IP?
2. Check the NSG attached to the Subnet.
3. Is the VM actually powered on and running?

---

## 6. Common Interview Questions

**Q: "Scenario: Share an encrypted golden AMI across accounts/regions (Dev → QA → Prod) — is it possible, and how?"**
*Answer:* Yes. The owning account can share an AMI with another account, but if it is encrypted, the KMS key policy also needs to permit the target account to use the key. For multi-region, I copy the AMI to the target region and configure the encrypted snapshot/key relationship there. I validate launch permissions, KMS permissions and the target account's ability to use the copied image before promoting it.

**Q: "Scenario: VPC peering is set up between two VPCs but instances still can’t communicate — what could be the issue?"**
*Answer:* VPC peering is a private point-to-point connection between two VPCs. It works well for a small number of non-transitive connections where route tables and CIDRs are easy to manage. If instances cannot communicate, I check overlapping CIDRs, route tables on both sides, Security Groups, NACLs, DNS and whether the instances are using the expected private addresses. I also confirm that the peering connection is active in both VPC route domains.

**Q: "Do you know about the OIDC provider in AWS?"**
*Answer:* An AWS OIDC provider is a trust configuration that lets IAM validate OIDC tokens from an identity provider. In EKS, the cluster exposes an OIDC issuer and this is used by IRSA so a Kubernetes ServiceAccount can exchange its projected token for temporary AWS credentials through STS. The value is fine-grained workload identity without distributing static AWS keys.

**Q: "What is the difference between Route 53 weighted and other routing policies, and which scenario would you use each in? How would you handle regional failover in an active-active setup?"**
*Answer:* Route 53 weighted routing distributes traffic according to configured weights, useful for canary or controlled traffic splits. Failover routing is health-check driven and fits active-passive designs well. Latency routing is useful when you want users sent toward lower-latency regions; geolocation/geoproximity address location-based policies. For active-active regional failover, I use health-aware routing such as latency or weighted records plus application/data replication so both regions can serve traffic safely.

---

## 7. Cheat Sheet

| Cloud Concept | AWS Equivalent | Azure Equivalent |
| :--- | :--- | :--- |
| **Virtual Server** | EC2 (Elastic Compute Cloud) | Virtual Machine (VM) |
| **Object Storage** | S3 (Simple Storage Service) | Blob Storage |
| **Managed K8s** | EKS (Elastic Kubernetes Service) | AKS (Azure Kubernetes Service) |
| **IaC/Deployment** | CloudFormation | ARM Templates / Bicep |
| **Identity/Auth** | IAM | Entra ID (formerly Azure AD) |
| **NoSQL Database** | DynamoDB | Cosmos DB |
| **Relational DB** | RDS / Aurora | Azure SQL Database |

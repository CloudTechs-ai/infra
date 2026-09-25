# zen-infra

 Terraform infrastructure for the **zen-pharma platform**, running on AWS with Kubernetes, RDS PostgreSQL, ECR, IAM, Secrets Manager, and GitHub Actions.

 The repository uses reusable Terraform modules and environment-specific configurations for **dev, QA, and production**.

 > **⚠️ Cost warning**
>
>  The dev environment includes EKS and a NAT Gateway, which can incur ongoing AWS charges. Destroy the environment when you're finished using it.

---

 ## 🏗️ Architecture

```
                              Internet
                                  │
                                  ▼
                        ┌──────────────────┐
                        │ Network Load      │
                        │ Balancer (NLB)    │
                        └────────┬─────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │ NGINX Ingress    │
                        │ Controller       │
                        └────────┬─────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
        ┌───────────┐      ┌────────────┐     ┌──────────────┐
        │ pharma-ui │      │ API Gateway│     │ Notification │
        │ React     │      │ :8080      │     │ :3000        │
        └───────────┘      └─────┬──────┘     └──────────────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
                 Auth       Catalog       Inventory
                 :8081       :8082          :8083
                    │            │            │
                    └────────────┼────────────┘
                                 │
                                 ▼
                       ┌──────────────────┐
                       │ RDS PostgreSQL   │
                       │ Private Subnets  │
                       └──────────────────┘

                 AWS Secrets Manager
                         │
                         ▼
              External Secrets Operator
                         │
                         ▼
                     Kubernetes
```

 ### Infrastructure overview

 | Component | Purpose |
| --- | --- |
| VPC | Isolated AWS network |
| Public subnets | NAT Gateway and load balancers |
| Private EKS subnets | Kubernetes worker nodes |
| Private RDS subnets | PostgreSQL database |
| EKS | Managed Kubernetes cluster |
| RDS PostgreSQL | Application database |
| ECR | Container image registry |
| IAM | AWS permissions and workload identity |
| Secrets Manager | Application secrets |
| S3 | Terraform remote state |
| GitHub Actions | Infrastructure CI/CD |

---

 # 📋 Table of Contents

 - Architecture
- Prerequisites
- Repository Structure
- AWS Resources
- Getting Started
  - 1\. Configure AWS
  - 2\. Create the Terraform State Bucket
  - 3\. Fork and Clone
  - 4\. Configure Your Account
  - 5\. Configure GitHub Secrets
  - 6\. Configure GitHub Environment
  - 7\. Deploy
  - 8\. Verify
- Environment Structure
- Networking
- Security
- CI/CD
- Day-2 Operations
- Destroying Infrastructure
- Troubleshooting
- Cost Estimate

---

 # 🔧 Prerequisites

 Install the following tools:

 | Tool | Version |
| --- | --- |
| Terraform | 1.10+ |
| AWS CLI | 2.x |
| Git | 2.x |
| kubectl | Latest recommended |

Verify your installation:

```
terraform version
aws --version
git --version
kubectl version --client
```

 You will also need:

 - An AWS account
- AWS permissions sufficient to create the infrastructure
- A GitHub account
- A fork of this repository

---

 # 📁 Repository Structure

```
zen-infra/
│
├── .github/
│   └── workflows/
│       └── terraform.yml
│
├── envs/
│   ├── dev/
│   │   ├── backend.tf
│   │   ├── providers.tf
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── qa/
│   │   └── ...
│   │
│   └── prod/
│       └── ...
│
├── modules/
│   ├── vpc/
│   ├── eks/
│   ├── rds/
│   ├── ecr/
│   ├── iam/
│   └── secrets-manager/
│
└── README.md
```

 ### Module responsibilities

 | Module | Responsibility |
| --- | --- |
| `vpc` | VPC, subnets, routes, NAT Gateway, Internet Gateway |
| `eks` | EKS cluster, node group, OIDC provider |
| `rds` | PostgreSQL, subnet group, security groups |
| `ecr` | Container repositories and lifecycle policies |
| `iam` | IAM roles and workload identity |
| `secrets-manager` | Application secrets |

Each environment calls the same modules with environment-specific configuration.

---

 # ☁️ AWS Resources

 The dev environment creates approximately:

 ### Networking

 - VPC: `10.0.0.0/16`
- 2 public subnets
- 2 private EKS subnets
- 2 private RDS subnets
- Internet Gateway
- NAT Gateway
- Route tables

 ### EKS

 - Kubernetes `1.33`
- EKS cluster
- Managed node group
- `t3.small` worker nodes
- Desired nodes: 3
- Minimum nodes: 2
- Maximum nodes: 4
- OIDC provider for IRSA

 ### RDS

 - PostgreSQL `15.7`
- Instance: `db.t3.micro`
- Storage: 20 GB
- Encrypted storage
- Private access only
- Port `5432`
- Accessible from the EKS security group

 ### ECR

 Container repositories for:

```
api-gateway
auth-service
drug-catalog-service
inventory-service
supplier-service
manufacturing-service
notification-service
pharma-ui
```

 Repositories use:

 - Image scanning on push
- Mutable image tags
- Lifecycle policies
- Automatic cleanup of older images

---

 # 🚀 Getting Started

 ## 1\. Configure AWS

 Configure the AWS CLI:

```
aws configure
```

 Use:

```
AWS Access Key ID:     <your-access-key>
AWS Secret Access Key: <your-secret-key>
Default region:        us-east-1
Default output:        json
```

 Verify your credentials:

```
aws sts get-caller-identity
```

 You should receive your AWS account and IAM identity information.

 > **Security:** Never commit AWS credentials to Git.

---

 ## 2\. Create the Terraform State Bucket

 Terraform state is stored remotely in S3.

 Choose a globally unique bucket name:

```
export TF_STATE_BUCKET="zen-pharma-terraform-state-YOUR-GITHUB-USERNAME"
```

 Create the bucket:

```
aws s3api create-bucket \
  --bucket "$TF_STATE_BUCKET" \
  --region us-east-1
```

 Enable versioning:

```
aws s3api put-bucket-versioning \
  --bucket "$TF_STATE_BUCKET" \
  --versioning-configuration Status=Enabled
```

 Enable encryption:

```
aws s3api put-bucket-encryption \
  --bucket "$TF_STATE_BUCKET" \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

 Block public access:

```
aws s3api put-public-access-block \
  --bucket "$TF_STATE_BUCKET" \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

 Verify:

```
aws s3 ls "s3://$TF_STATE_BUCKET"
```

 The bucket should exist and initially be empty.

---

 ## 3\. Fork and Clone

 Fork the repository to your GitHub account.

 Then:

```
git clone https://github.com/YOUR-GITHUB-USERNAME/zen-infra.git
cd zen-infra
```

---

 ## 4\. Configure Your Account

 Update the S3 bucket in:

```
envs/dev/backend.tf
envs/qa/backend.tf
envs/prod/backend.tf
```

 Example:

```
terraform {
  backend "s3" {
    bucket       = "zen-pharma-terraform-state-YOUR-GITHUB-USERNAME"
    key          = "envs/dev/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

 Each environment should use its own state key:

```
envs/dev/terraform.tfstate
envs/qa/terraform.tfstate
envs/prod/terraform.tfstate
```

 ### Configure GitHub organization/user

 Update `github_org` in:

```
envs/dev/variables.tf
envs/qa/variables.tf
envs/prod/variables.tf
```

 Example:

```
variable "github_org" {
  description = "GitHub username or organization"
  type        = string
  default     = "YOUR-GITHUB-USERNAME"
}
```

---

 # 🔐 5. Configure GitHub Secrets

 Go to:

 **GitHub → Repository → Settings → Secrets and variables → Actions**

 Create these repository secrets:

 | Secret | Purpose |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | AWS authentication |
| `AWS_SECRET_ACCESS_KEY` | AWS authentication |
| `DEV_DB_PASSWORD` | RDS master password |
| `DEV_JWT_SECRET` | JWT signing secret |

Generate a strong JWT secret:

```
openssl rand -hex 32
```

 > **Important:** Secrets should never be committed to Git or stored in Terraform files.

---

 # 🛡️ 6. Configure GitHub Environment

 Create a GitHub environment named:

```
dev
```

 Navigate to:

 **Settings → Environments → New environment**

 Add a required reviewer under:

 **Deployment protection rules → Required reviewers**

 This creates a manual approval gate before Terraform can apply infrastructure changes.

---

 # 🚢 7. Deploy

 The recommended workflow is:

```
Feature branch
      │
      ▼
Pull Request
      │
      ▼
Terraform Plan
      │
      ▼
Review
      │
      ▼
Merge to main
      │
      ▼
Fresh Terraform Plan
      │
      ▼
Manual Approval
      │
      ▼
Terraform Apply
```

 Create a feature branch:

```
git checkout -b feature/initial-setup
```

 Make a small configuration change, then:

```
git add .
git commit -m "feat: initial infrastructure setup"
git push origin feature/initial-setup
```

 Open a Pull Request against `main`.

 The GitHub Actions pipeline will run:

```
terraform fmt
terraform init
terraform validate
terraform plan
```

 Review the plan before merging.

 After merging, the pipeline generates another plan and waits for deployment approval.

 Go to:

 **GitHub → Actions → Terraform Infrastructure → Review deployments**

 Approve the deployment.

 ### Expected deployment time

 A full dev deployment typically takes approximately:

 - EKS cluster: \~10 minutes
- EKS node group: \~5 minutes
- RDS: \~5 minutes

 Allow approximately **15–25 minutes** for the complete deployment.

---

 # ✅ 8. Verify the Infrastructure

 After Terraform finishes, check the outputs:

```
cd envs/dev
terraform output
```

 You should see values such as:

```
eks_cluster_name
rds_endpoint
```

 ### Check EKS

```
aws eks describe-cluster \
  --name pharma-dev-cluster \
  --query 'cluster.status'
```

 Expected:

```
"ACTIVE"
```

 Configure kubectl:

```
aws eks update-kubeconfig \
  --region us-east-1 \
  --name pharma-dev-cluster
```

 Check nodes:

```
kubectl get nodes
```

 Expected:

```
NAME                    STATUS   ROLES    AGE
...                     Ready    <none>   ...
...                     Ready    <none>   ...
...                     Ready    <none>   ...
```

 Check namespaces:

```
kubectl get namespaces
```

---

 # 🌐 Networking

 The dev environment uses:

 | Subnet | CIDR | AZ | Purpose |
| --- | --- | --- | --- |
| Public 1 | `10.0.1.0/24` | `us-east-1a` | NAT / Load Balancer |
| Public 2 | `10.0.2.0/24` | `us-east-1b` | NAT / Load Balancer |
| Private EKS 1 | `10.0.3.0/24` | `us-east-1a` | EKS nodes |
| Private EKS 2 | `10.0.4.0/24` | `us-east-1b` | EKS nodes |
| Private RDS 1 | `10.0.5.0/24` | `us-east-1a` | PostgreSQL |
| Private RDS 2 | `10.0.6.0/24` | `us-east-1b` | PostgreSQL |

Worker nodes and RDS remain in private subnets.

 Outbound traffic from private resources uses the NAT Gateway.

---

 # 🔒 Security Design

 The infrastructure is designed around several security principles.

 ### Private worker nodes

 EKS worker nodes are deployed in private subnets and don't receive public IP addresses.

 ### Private database

 RDS is deployed in private subnets and isn't publicly accessible.

 Port `5432` is restricted to the appropriate EKS security group.

 ### Secrets Manager

 Application secrets are stored in AWS Secrets Manager:

```
/pharma/dev/db-credentials
/pharma/dev/jwt-secret
```

 Secrets are not stored in Git or Kubernetes manifests.

 ### External Secrets Operator

 ESO retrieves secrets from Secrets Manager and makes them available to workloads inside Kubernetes.

 ### IRSA

 Kubernetes workloads can use IAM Roles for Service Accounts instead of static AWS credentials.

 ### GitHub Actions

 The repository uses GitHub Actions to automate infrastructure deployment.

 > **Production improvement:** Replace long-lived AWS access keys in GitHub Secrets with GitHub Actions OIDC wherever possible.

---

 # 🔄 CI/CD

 The infrastructure pipeline is defined in:

```
.github/workflows/terraform.yml
```

 ### Pull Request

 Every infrastructure PR should run:

```
terraform fmt -check
terraform init
terraform validate
terraform plan
```

 The plan should be reviewed before merging.

 ### Main branch

 After merging:

```
Terraform Plan
      ↓
Manual Approval
      ↓
Terraform Apply
```

 This provides a controlled deployment process while keeping infrastructure changes version-controlled.

---

 # 🛠️ Day-2 Operations

 ## Make an infrastructure change

 Create a branch:

```
git checkout -b feature/update-infrastructure
```

 Make your Terraform changes.

 Test locally:

```
cd envs/dev

terraform init

terraform plan \
  -var="db_password=dummy" \
  -var="jwt_secret=dummy"
```

 Commit and push:

```
git add .
git commit -m "feat: update infrastructure"
git push origin feature/update-infrastructure
```

 Open a Pull Request and review the generated Terraform plan.

---

 ## Check Terraform state

```
cd envs/dev
terraform state list
```

 Inspect a resource:

```
terraform state show module.eks.aws_eks_cluster.main
```

---

 ## Check for drift

 To compare Terraform state with the actual infrastructure:

```
terraform plan
```

 For a refresh-only check:

```
terraform plan -refresh-only
```

 This is particularly useful after manually changing or deleting AWS resources.

---

 ## Scale the EKS node group

 Update the module configuration:

```
module "eks" {
  # ...

  desired_capacity = 5
  min_size         = 3
  max_size         = 8
}
```

 Then use the normal PR workflow.

---

 # 💥 Destroying Infrastructure

 > **⚠️ WARNING**
>
>  Destroying the environment permanently removes infrastructure. This includes EKS, RDS, networking resources, and other AWS resources. Database data may be permanently lost.

 ## Recommended: GitHub Actions

 Use the infrastructure workflow:

 **GitHub → Actions → Terraform Infrastructure → Run workflow**

 Select:

```
Terraform action: destroy
```

 Enter the required confirmation:

```
destroy
```

 The destroy job will require approval before execution.

 Allow approximately **15–25 minutes** for a complete teardown.

---

 ## Local destroy

 From the dev environment:

```
cd envs/dev

terraform init
```

 Then:

```
terraform destroy \
  -var="db_password=dummy" \
  -var="jwt_secret=dummy" \
  -var="github_org=YOUR-GITHUB-USERNAME"
```

 Terraform will show the resources it intends to delete.

 Type:

```
yes
```

 to confirm.

 ### Important

 Don't manually delete AWS resources while Terraform is actively destroying them.

 If resources have already been manually deleted, refresh Terraform's state first:

```
terraform plan -refresh-only
```

 Then:

```
terraform apply -refresh-only
```

 Finally:

```
terraform plan
```

---

 # 🧹 Removing the Terraform State Bucket

 `terraform destroy` does **not** delete the S3 backend bucket because Terraform needs the backend to manage its own state.

 After all environments have been destroyed, you can remove the bucket manually.

 Empty it:

```
aws s3 rm \
  s3://zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --recursive
```

 Then delete it:

```
aws s3api delete-bucket \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --region us-east-1
```

 > Make sure you no longer need any Terraform state versions before deleting the bucket.

---

 # 🐛 Troubleshooting

 ## `RepositoryAlreadyExistsException`

 ECR repositories may still contain images.

 Delete the repository:

```
aws ecr delete-repository \
  --repository-name api-gateway \
  --force \
  --region us-east-1
```

 Repeat for the affected repositories.

---

 ## Terraform says a resource already exists

 Refresh state:

```
terraform plan -refresh-only
```

 Then:

```
terraform apply -refresh-only
```

 If the resource genuinely exists in AWS but isn't in Terraform state, investigate before importing or deleting it.

---

 ## Terraform destroy says a subnet has dependencies

 This usually means something is still using the subnet.

 Check network interfaces:

```
aws ec2 describe-network-interfaces \
  --filters "Name=subnet-id,Values=<subnet-id>" \
  --query 'NetworkInterfaces[*].[NetworkInterfaceId,Status,Description,InterfaceType]' \
  --output table
```

 Common causes include:

 - NAT Gateway
- Load Balancer
- VPC Endpoint
- Lambda VPC ENI
- EKS resources
- Other AWS-managed network interfaces

 Delete the owning resource rather than manually deleting the ENI.

---

 ## Internet Gateway cannot be detached

 You may see:

```
DependencyViolation:
Network VPC has some mapped public address(es).
```

 Check public addresses:

```
aws ec2 describe-addresses \
  --filters "Name=domain,Values=vpc" \
  --query 'Addresses[*].[AllocationId,PublicIp,AssociationId,InstanceId,NetworkInterfaceId]' \
  --output table
```

 Also check NAT Gateways:

```
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=<vpc-id>" \
  --query 'NatGateways[*].[NatGatewayId,State,SubnetId]' \
  --output table
```

 Remove the resource owning the public address and then retry:

```
terraform destroy
```

---

 ## Terraform state lock error

 If you see:

```
Error acquiring the state lock
```

 First make sure another Terraform operation isn't running.

 If you're certain there isn't an active operation:

```
terraform force-unlock <LOCK_ID>
```

 > Never force-unlock a state that another Terraform operation is actively using.

---

 ## EKS nodes aren't joining

 Check:

```
kubectl get nodes
```

 Verify the node group IAM role has the appropriate permissions, including:

```
AmazonEKSWorkerNodePolicy
AmazonEKS_CNI_Policy
AmazonEC2ContainerRegistryReadOnly
```

 Also check the Terraform/GitHub Actions apply logs for errors during node group creation.

---

 ## Cannot connect to EKS

 Refresh kubeconfig:

```
aws eks update-kubeconfig \
  --region us-east-1 \
  --name pharma-dev-cluster
```

 Verify your AWS identity:

```
aws sts get-caller-identity
```

 Check cluster status:

```
aws eks describe-cluster \
  --name pharma-dev-cluster \
  --query 'cluster.status'
```

 Expected:

```
"ACTIVE"
```

---

 # 💰 Cost Estimate

 Approximate dev environment costs:

 | Resource | Approximate cost |
| --- | --- |
| EKS control plane | \~$0.10/hour |
| 3 × `t3.small` | \~$43/month |
| RDS `db.t3.micro` | \~$14/month |
| NAT Gateway | \~$32/month + data transfer |
| ECR | Minimal for small image storage |
| Secrets Manager | \~$0.80/month for 2 secrets |
| **Estimated total** | **\~$160–180/month** |

AWS pricing varies by region, usage, data transfer, storage, and configuration. Treat these numbers as estimates rather than billing guarantees.

 ### 💡 Tip for learners

 If you're not actively using the environment:

```
terraform destroy
```

 EKS and NAT Gateway are among the larger ongoing costs in this architecture.

---

 # 🎯 Interview Preparation

 This project is designed to demonstrate practical experience with:

 - Terraform
- Terraform modules
- Terraform state
- Remote S3 state
- Infrastructure as Code
- AWS VPC networking
- Amazon EKS
- Kubernetes
- RDS PostgreSQL
- Amazon ECR
- IAM
- IRSA
- OIDC
- AWS Secrets Manager
- GitHub Actions
- CI/CD
- Infrastructure drift
- Infrastructure lifecycle management

 Recommended interview topics include:

 ### Terraform

 - State management
- Remote backends
- Modules
- Variables and outputs
- Resource dependencies
- `terraform plan`
- `terraform apply`
- `terraform destroy`
- Drift detection
- State locking
- Importing existing resources
- Workspaces vs separate environments

 ### AWS

 - VPC design
- Public vs private subnets
- NAT Gateway
- Internet Gateway
- Route tables
- Security groups
- EKS networking
- IAM
- OIDC
- IRSA
- RDS networking

 ### CI/CD

 - GitHub Actions
- Pull-request plans
- Deployment approvals
- Secrets management
- OIDC authentication
- Infrastructure deployment workflows

---

 # 🗺️ Environment Strategy

 The repository supports three isolated environments:

```
envs/
├── dev/
├── qa/
└── prod/
```

 Each environment:

 - Has its own Terraform configuration
- Has its own remote state key
- Uses the same reusable modules
- Can have different resource sizing
- Can have different security and availability settings

 Example:

```
                ┌───────────────┐
                │ Shared Modules│
                └───────┬───────┘
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
        DEV            QA           PROD
          │             │             │
       State          State         State
       Object         Object        Object
```

---

 # 📌 Design Principles

 This project follows several core infrastructure principles:

 1. **Infrastructure as Code** — AWS infrastructure is defined in Terraform.
2. **Reusable modules** — environments share common Terraform modules.
3. **Remote state** — Terraform state is stored securely in S3.
4. **Environment isolation** — dev, QA, and production use separate state.
5. **Private workloads** — EKS nodes and RDS are deployed in private subnets.
6. **Secrets outside Git** — sensitive values are stored in AWS Secrets Manager.
7. **Least privilege** — IAM permissions should be scoped to required resources.
8. **Automated validation** — Terraform plans run through GitHub Actions.
9. **Human approval** — production infrastructure changes should require explicit approval.
10. **Repeatability** — environments can be recreated from code.

---

 ## 📚 Project Documentation

 Additional documentation can be found in the `docs/` directory:

```
docs/
├── terraform-interview-questions.md
└── github-actions-interview-questions.md
```

 The Terraform interview guide covers core Terraform concepts, state management, modules, CI/CD, and real-world scenarios based on this project.

---

 ## 🧰 Technology Stack

 | Technology | Purpose |
| --- | --- |
| Terraform 1.10+ | Infrastructure as Code |
| AWS | Cloud platform |
| Amazon VPC | Networking |
| Amazon EKS | Kubernetes |
| Kubernetes | Container orchestration |
| Amazon RDS | PostgreSQL database |
| Amazon ECR | Container registry |
| AWS IAM | Identity and access management |
| AWS Secrets Manager | Secrets management |
| Amazon S3 | Terraform state |
| GitHub Actions | CI/CD |
| NGINX Ingress | Kubernetes ingress |

---

 ## 👤 Project

 **zen-pharma infrastructure**

 Built with:

 **Terraform · AWS · EKS · Kubernetes · RDS · ECR · IAM · Secrets Manager · GitHub Actions**

 > Infrastructure should be reproducible, reviewable, and disposable.
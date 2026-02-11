# Terraform EKS Cluster with Managed Node Group

This repository contains Terraform code to provision an **Amazon EKS Cluster** along with an **EKS Managed Node Group**.  
It includes IAM roles, required AWS-managed policies, and auto-scaling worker nodes.

---

## Prerequisites

Before you begin, ensure you have:

- AWS CLI installed and configured
- Terraform installed (v1.3+ recommended)
- An existing VPC with **at least two subnets**
- IAM permissions for:
  - EKS
  - EC2
  - IAM

### Configure AWS CLI
```bash
aws configure
```

Verify Terraform:
```bash
terraform -v
```

### Project Structure
```text
EKS-NODE/
├── .terraform/
├── .terraform.lock.hcl
├── main.tf
├── provider.tf
├── variable.tf
├── output.tf
├── terraform.tfstate
├── terraform.tfstate.backup
└── README.md
```
### Step 1: Create Project Directory & Files
```bash
mkdir EKSNODE
cd EKSNODE
touch main.tf provider.tf variable.tf output.tf README.md
```
### Step 2: Add the following Infrastructure content in the files

In AWS Provider (provider.tf)

```hcl
provider "aws" {
  region = "us-east-1"
}
```

In Variables (variable.tf)

```hcl
variable "subnet_ids" {
  description = "List of subnet IDs for the EKS cluster"
  type        = list(string)
  default     = ["subnet-123456789986765", "subnet-12345678998765"] #Add subnets attached to the VPC below
}

variable "vpc_id" {
  default = "vpc-123456789009876" #Add your own VPC
}
```

In Main file (main.tf)

```hcl
## Create an EKS cluster with a node group
resource "aws_eks_cluster" "cluster_eks" {
  name = "eks-clusterwithnode"
  role_arn = aws_iam_role.cluster.arn ## IAM role for the EKS cluster
  vpc_config {
    subnet_ids = var.subnet_ids ## List of subnet IDs for the EKS cluster
  }

  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy,
  ] ## Ensure the IAM role policy attachment is created before the EKS cluster

}

resource "aws_iam_role" "cluster" {
  name = "eks-cluster-role-nodegrp"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "sts:AssumeRole",
          "sts:TagSession"
        ]
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
      },
    ]
  })
}

## Attach the AmazonEKSClusterPolicy to the IAM role for the EKS cluster
resource "aws_iam_role_policy_attachment" "cluster_AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy" 
  role       = aws_iam_role.cluster.name
}

## Create a node group for the EKS cluster
## Specify the instance type and AMI type for the instances in the node group
resource "aws_eks_node_group" "node_group_eks" {
  cluster_name    = aws_eks_cluster.cluster_eks.name
  node_group_name = "node-group"
  node_role_arn   = aws_iam_role.node_group.arn
  subnet_ids      = var.subnet_ids
  instance_types = [ "t3.micro" ] 
  ami_type = "AL2_x86_64_STANDARD"
  scaling_config {
    desired_size = 1
    max_size     = 2
    min_size     = 1
  }

  update_config {
    max_unavailable = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.example-AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.example-AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.example-AmazonEC2ContainerRegistryReadOnly,
    aws_iam_role_policy_attachment.example-AmazonEKSWorkerNodeMinimalPolicy
  ]
}

## Create an IAM role for the EKS node group and attach necessary policies
resource "aws_iam_role" "node_group" {
  name = "eks-node-group"

  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
    Version = "2012-10-17"
  })
}

resource "aws_iam_role_policy_attachment" "example-AmazonEKSWorkerNodePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.node_group.name
}

resource "aws_iam_role_policy_attachment" "example-AmazonEKS_CNI_Policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.node_group.name
}

resource "aws_iam_role_policy_attachment" "example-AmazonEC2ContainerRegistryReadOnly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.node_group.name
}

resource "aws_iam_role_policy_attachment" "example-AmazonEKSWorkerNodeMinimalPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodeMinimalPolicy"
  role       = aws_iam_role.node_group.name
}
```

### Step 3: Initialize Terraform

```bash
terraform init
```
### Step 4: Apply Terraform (Create EKS and Node Groups)

```bash
terraform apply
```










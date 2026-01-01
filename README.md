# Building a Complete DevSecOps Project

![1_50Xa4Lct9GaJBhUsuN337g](https://github.com/user-attachments/assets/47c518ce-bc73-4f7e-9aef-85992975c010)

## (Part 1) — Automating AWS Infrastructure with Terraform Cloud & GitHub Actions

So I decided to do it the right way — the DevOps way.
This series is my step-by-step journey of building a Netflix-clone application with a full DevSecOps pipeline — everything from infrastructure as code, security scans, and CI/CD automation to monitoring.

And we’re starting right at ground zero — setting up Terraform Cloud for remote state management and using GitHub Actions to automate infrastructure provisioning on AWS.

### Objective
In Part 1, the goal is simple but powerful:
build and manage AWS infrastructure using Terraform Cloud — no local state headaches, no manual apply, and no “oops, I deleted my .tfstate file” moments.

We’ll also integrate GitHub Actions to trigger Terraform runs automatically whenever we push code — setting the stage for a fully automated infrastructure pipeline.

By the end of this part, you’ll have:
  - A working Terraform Cloud workspace linked with GitHub
  - Automated infra provisioning via GitHub Actions
  - AWS EC2 instances ready for upcoming Jenkins, monitoring, and Kubernetes setups

### Terraform Scripts for EC2 and other AWS Services(IAM, VPC,etc)

#### backend.tf 
```hcl
terraform {
  required_version = ">= 1.13.3"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.14.1"
    }
  }

  cloud {}
}
provider "aws" {
  region = var.aws-region
}
```
#### gather.tf
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name = "name"
    values = [
      "ubuntu/images/hvm-ssd/ubuntu-noble-24.04-amd64-server-*",
      "ubuntu/images/hvm/ubuntu-noble-24.04-amd64-server-*",
      "ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-*",
      "ubuntu/images/hvm-ebs-gp3/ubuntu-noble-24.04-amd64-server-*"
    ]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"]
}
```
#### iam.tf
```hcl
resource "aws_iam_role" "role" {
  name = "${local.org}-${local.project}-${local.env}-ssm-iam-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-ssm-iam-role"
    Env  = "${local.env}"
  }
}

resource "aws_iam_role_policy_attachment" "ssm_managed_policy" {
  role       = aws_iam_role.role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "iam-instance-profile" {
  name = "${local.org}-${local.project}-${local.env}-instance-profile"
  role = aws_iam_role.role.name
}
```
#### vpc.tf
```hcl
locals {
  org     = "damir"
  project = "netflix-clone"
  env     = var.env
}

resource "aws_vpc" "vpc" {
  cidr_block           = var.cidr-block
  instance_tenancy     = "default"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-vpc"
    Env  = "${local.env}"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-igw"
    env  = var.env
  }

  depends_on = [aws_vpc.vpc]
}

resource "aws_subnet" "public-subnet" {
  count                   = var.pub-subnet-count
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = element(var.pub-cidr-block, count.index)
  availability_zone       = element(var.pub-availability-zone, count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-public-subnet-${count.index + 1}"
    Env  = var.env
  }

  depends_on = [aws_vpc.vpc]
}


resource "aws_route_table" "public-rt" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-public-route-table"
    env  = var.env
  }

  depends_on = [aws_vpc.vpc]
}

resource "aws_route_table_association" "public-rta" {
  count          = 4
  route_table_id = aws_route_table.public-rt.id
  subnet_id      = aws_subnet.public-subnet[count.index].id

  depends_on = [aws_vpc.vpc,
    aws_subnet.public-subnet
  ]
}

resource "aws_security_group" "default-ec2-sg" {
  name        = "${local.org}-${local.project}-${local.env}-sg"
  description = "Default Security Group"

  vpc_id = aws_vpc.vpc.id

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"] // It should be specific IP range
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-sg"
  }
}
```
#### main.tf
```hcl
locals {
  instance_names = [
    "jenkins-server",
    "monitoring-server",
    "kubernetes-master-node",
    "kubernetes-worker-node"
  ]
}

resource "aws_instance" "ec2" {
  count                  = var.ec2-instance-count
  ami                    = data.aws_ami.ubuntu.id
  subnet_id              = aws_subnet.public-subnet[count.index].id
  instance_type          = var.ec2_instance_type[count.index]
  iam_instance_profile   = aws_iam_instance_profile.iam-instance-profile.name
  vpc_security_group_ids = [aws_security_group.default-ec2-sg.id]
  root_block_device {
    volume_size = var.ec2_volume_size
    volume_type = var.ec2_volume_type
  }

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-${local.instance_names[count.index]}"
    Env  = "${local.env}"
  }
}
```
#### variables.tf
```hcl
variable "aws-region" {}
variable "env" {}
variable "cidr-block" {}
variable "pub-subnet-count" {}
variable "pub-cidr-block" {
  type = list(string)
}
variable "pub-availability-zone" {
  type = list(string)
}
variable "ec2-instance-count" {}
variable "ec2_instance_type" {
  type = list(string)
}
variable "ec2_volume_size" {}
variable "ec2_volume_type" {}
```
#### dev.auto.tfvars
```hcl
aws-region            = "us-east-1"
env                   = "dev"
cidr-block            = "10.0.0.0/16"
pub-subnet-count      = 4
pub-cidr-block        = ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20", "10.0.64.0/20"]
pub-availability-zone = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d"]
ec2-instance-count    = 4
ec2_instance_type     = ["t3a.xlarge", "t3a.medium", "t3a.medium", "t3a.medium"]
ec2_volume_size       = 50
ec2_volume_type       = "gp3"
```
### Setting Up Terraform Cloud Terraform State Management
Click on the URL to signin/signup for Terraform Cloud
```bash
https://app.terraform.io/session
```
Click on Continue with HCP account

<img width="681" height="334" alt="Screenshot 2026-01-01 at 12 49 26 PM" src="https://github.com/user-attachments/assets/780635be-45b3-43e3-a66e-be2cc04ddc20" />

We are creating an account on HashiCorp Cloud Platform, as it covers multiple services along with Terraform . This will be useful for your future work. 
If you want, you can also simply sign up for a Terraform account.
Now, Click on Sign up

<img width="512" height="703" alt="Screenshot 2026-01-01 at 12 50 43 PM" src="https://github.com/user-attachments/assets/f258ea97-9a47-446b-a0e1-21ee45672cbc" />

Now, I will click on continue with GitHub, as I already have a GitHub Account.

Once you set up your account, it will look like the snippet below.
Although I have already created one organisation.
Now, we will create our first organisation. So, click on Create organization the " Show " on the top right

Press enter or click to view image in full size

![0_9zRuqftZaI8gqU8y](https://github.com/user-attachments/assets/44f58d3f-cc7c-4213-bd36-9079d124c29d)

To create an organisation, you have to provide three information. Follow the below things:
Type of Organisation: If you are using it for a personal use case, go for Personal; else, Business is fine, but you have to pay
Organisation Name: It must be a unique name.
Email address: Provide your email address, and click on Next.

![0_ewlIQBFvyF73FHku](https://github.com/user-attachments/assets/97d30d41-2035-4038-8d8e-714681e5f6b2)

Once you create the Organization , It will take you to the workspaces section.

Now, we will create a workspace

To create a workspace, click on create a workspace

Press enter or click to view image in full size

![0_sRHXhScB6xtW681A](https://github.com/user-attachments/assets/1cdaa8ca-f60e-4259-b25f-1407ed828af5)

Now, we have to provide our AWS credentials to Terraform Workspace as our plan, and other Terraform operations will be running on HashiCorp-hosted servers.
Make sure to create the Environment Variables with the correct Keys and values under the Variables section.

- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

![1_4OPl_5Sf6bCPFrusQRqP4A](https://github.com/user-attachments/assets/9c4e3be8-62e0-4511-90a7-247038c6990f)

Now, we will have to create the token to authenticate with Terraform Cloud from GitHub Actions while running Terraform operations

Go to your Profile’s Account settings in the top right

After that, click on Tokens, then click on Create an API token

![1_QKc8Bp0QXlCTe00gDwL9MQ](https://github.com/user-attachments/assets/18f16b35-a206-4ff3-8c96-bcde89720e6d)

Provide the description and set the expiration of your token, and click on Generate token

![1_2GPPMj2ks_yxfQ4B2dSwUw](https://github.com/user-attachments/assets/284650ca-47b6-49a6-a692-c469d6ec7f0c)

### Automating Infrastructure using Terraform with GitHub Actions
Now, we will automate our AWS Infrastructure using GitHub Actions

The workflow script written below
#### terraform.yaml
```yaml
name: '🔨 Infrastructure Configuring using Terraform on AWS' 

on: 
  workflow_dispatch:
    inputs:
      action:
        type: choice
        description: 'Terraform Action'
        options:
          - plan
          - apply
          - destroy
        required: true
        default: 'plan'

env:
  TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
  TF_WORKSPACE: ${{ secrets.TF_WORKSPACE }}
  TF_CLOUD_ORGANIZATION: ${{ secrets.TF_CLOUD_ORGANIZATION }}
permissions:
  contents: read


jobs:
  infrastructure-setup-using-terraform:
    name: Terraform ${{ github.event.input.actions }}
    runs-on: ubuntu-24.04
    environment: production

    defaults:
      run:
        shell: bash
        working-directory: Terraform

    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Setting Up the Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.13.3"
          terraform_wrapper: true
          cli_config_credentials_token: ${{ secrets. TF_API_TOKEN }}

      - name: Cache Terraform
        uses: actions/cache@v4
        with:
          path: |
            ~/.terraform.d/plugin-cache
            .terraform
          key: ${{ runner.os }}-terraform-${{ hashFiles('**/*.tf') }}
          restore-keys: |
            ${{ runner.os }}-terraform-
          
      - name: Initialising the Terraform
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check --diff

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        if: ${{ github.event.inputs.action == 'plan' }}
        run: terraform plan -input=false

      - name: Terraform Apply
        if: ${{ github.event.inputs.action == 'apply' }}
        run: terraform apply -auto-approve

      - name: Terraform destroy
        if: ${{ github.event.inputs.action == 'destroy' }}
        run: terraform destroy -auto-approve
```
Now, we need to create three secrets for Terraform Cloud in the GitHub Repo.
  - Navigate to your Project Repo -> Settings
  - Expand the Secrets and variables, then click on Actions

![1_ioqNIvaos704LRsEEDoRHQ](https://github.com/user-attachments/assets/66ee0182-7d68-416e-8ee3-4c1931acf57a)


Then, add the three variables with correct values
  - TF_API_TOKEN
  - TF_CLOUD_ORGANIZATION
  - TF_WORKSPACE

![1_KmR4LFhOcmbwlXLdRrjWMQ](https://github.com/user-attachments/assets/2fd3003b-1413-459f-ba1e-63c444b40230)

#### Now, we are ready to deploy our Infrastructure using Terraform, Terraform Cloud, and GitHub Actions on AWS
Go to the Actions of the GitHub Repository

<img width="1900" height="933" alt="Screenshot 2026-01-01 at 11 58 27 AM" src="https://github.com/user-attachments/assets/14eb6be2-668f-44a9-8553-131f808d1cad" />

Apply is successful

![1_iGAFZGVCH8y8TnJCpPNwSg](https://github.com/user-attachments/assets/cf519332-411a-45f8-9c87-08517d8b5426)

You can validate the resource from Terraform Cloud as well

<img width="1444" height="630" alt="Screenshot 2026-01-01 at 12 03 22 PM" src="https://github.com/user-attachments/assets/236cb423-3bca-47a3-ac2d-50182ef1eeed" />

Now, let’s go to the AWS Console and see whether our EC2s are there or not

<img width="1607" height="308" alt="Screenshot 2026-01-01 at 12 02 45 PM" src="https://github.com/user-attachments/assets/00c05119-d1de-4f29-b857-350fdd49235c" />



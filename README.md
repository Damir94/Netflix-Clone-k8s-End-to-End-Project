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

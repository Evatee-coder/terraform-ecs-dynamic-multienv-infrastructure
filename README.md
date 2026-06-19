# terraform-ecs-dynamic-multienv-infrastructure
multi-environment deployment with Terraform for ECS applications, focusing on dynamic code, real-time autoscaling, and managing configurations via variables and locals

# Project Overview
This project provisions a complete, production-grade containerised application platform on AWS using Terraform. It deploys a Dockerised student portal application onto ECS Fargate, backed by a managed PostgreSQL database (RDS), puts an Application Load Balancer in front of it with HTTPS enabled, and makes it accessible via a proper custom Route 53 subdomain.
The infrastructure is designed for multi-environment deployment. The same Terraform codebase targets both dev and prod environments without any code duplication, driven entirely by per-environment .tfvars and .tfbackend files. Dynamic Terraform patterns which includes: list indexing, map variables, for_each, and string interpolation eliminate hard-coded values and make every resource name, size, and connection string environment-aware. Real-time ECS autoscaling is wired to CPU utilisation metrics, which allows the service to scale between one and five tasks automatically under load.


## Architecture

![Architecture Diagram](images/victor-adetayo-aws-architecture-dark.drawio.png)

# Prerequisites
## Required Tools

| Tool | Minimum Version | Notes |
|---|---|---|
| Terraform | 1.13.3 (exact) | developer.hashicorp.com/terraform/install |
| AWS CLI | 2.x | Required for credential configuration and ECR login |
| Docker | Any recent | Required only to build and push the application image to ECR |

# AWS Account Requirements

- An AWS account with programmatic access set up
- A Route 53 public hosted zone for your domain. Update domain_name in vars/{env}.tfvars if using your own
- A customer-managed KMS key created with alias alias/dev-kms (dev) and alias/prod-kms (prod) in us-east-1 before running terraform apply.
- An S3 bucket with versioning enabled for remote state (update the backend configs for remote state storage)

# How to Run
## Step 1 - Clone the repo
    git clone https://github.com/Evatee-coder/terraform-ecs-dynamic-multienv-infrastructure.git
    cd terraform-ecs-dynamic-multienv-infrastructure/advance-terraform

## Step 2 - Configure AWS credentials
    aws configure
    Enter AWS Access Key ID, Secret Access Key, and set region to your own

## Step 3 -Push the application Docker Image to ECR
Reference the ECS task definition student-portal:1.0 in your account's ECR. Build and push the image before applying Terraform, or the ECS tasks will fail to start.

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
 
# Tag and push
docker tag student-portal:1.0 \
  <YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/student-portal:1.0
 
docker push \
  <YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/student-portal:1.0

# Step 4 -INitialize Terraform with the environment backend
Each environment has its own remote state. Pass the corresponding .tfbackend file to terraform init.
    # For dev
    terraform init -backend-config=vars/dev.tfbackend
 
    # For prod
    terraform init -backend-config=vars/prod.tfbackend

# Step 5 — Review the execution plan
    # For dev
    terraform plan -var-file=vars/dev.tfvars
    
    # For prod
    terraform plan -var-file=vars/prod.tfvars

# Step 6 — Apply the infrastructure
Type yes when prompted. A full fresh apply takes approximately 10-15 minutes; RDS provisioning is the longest step.
    # For dev
    terraform apply -var-file=vars/dev.tfvars
    
    # For prod
    terraform apply -var-file=vars/prod.tfvars

# Autoscaling Configuration
ECS service autoscaling is configured in advance-terraform/ecs.tf using AWS Application Auto Scaling with a Target Tracking Scaling policy.

# Load-testing autoscaling
Use the following commands to simulate load and observe autoscaling in action:
    # 1000 requests, 200 concurrent workers
    docker run --rm williamyeh/hey \
    -n 1000 -c 200 \
    https://dev.<your-subdomain>.<your-domain>.com/login
    
    # Alternative load generator
    docker run fjudith/load-test \
    -h https://dev.<your-subdomain>.<your-domain>.com/login \
    -c 10 -r 1000

Monitor the ECS service in the AWS Console (ECS -> Cluster -> Service -> Metrics tab) to observe the task count adjusting in real time.

# Outputs

## Screenshots
The screenshots below confirm the end-to-end deployment: DNS resolving correctly, TLS certificate active, the ALB routing traffic to ECS, and the application successfully connecting to RDS.
**Live Application — https://dev.studentportal.eva-tee.com/login**

![Live Application](images/studentportal-login.png)

# Clean Up
Destroy all resources for an environment when they are no longer needed. This action is irreversible — RDS will create a final snapshot before deletion.
    # Destroy dev
    terraform init -backend-config=vars/dev.tfbackend
    terraform destroy -var-file=vars/dev.tfvars
    
    # Destroy prod
    terraform init -reconfigure -backend-config=vars/prod.tfbackend
    terraform destroy -var-file=vars/prod.tfvars

**Note**: The RDS instance has skip_final_snapshot = false. Terraform will create a snapshot named mydb-final-snapshot-{YYYY-MM-DD-hhmm} before deleting the instance. Delete this snapshot manually from the RDS console if it is no longer needed to avoid ongoing storage charges.



# Troubleshooting
Below is the summary of all errors and their fixes at a glance:

| # | Error | Fix location |
|---|---|---|
| #1 | .terraform/ accidentally commited to git | Add .terraform/ to .gitignore, run git rm --cached|
| #2 | CannotPullContainerError — wrong ECR name | Fix image URI in advance-terraform/ecs.tf|
| #3 | ECS tasks cannot reach ECR (no NAT route) | Add aws_route.private_nat_route in network.tf |
| #4 | NAT Gateway EIP allocation ID not found | Replace aws_eip data source with resource in network.tf |
| #5 | DB_LINK URL broken by special characters | Set special = false on random_password in rds.tf|




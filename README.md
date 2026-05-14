# aws-three-tier-terraform-private

Below is a complete documentation section for what we worked on today. Replace each screenshot path with your actual screenshot file name after uploading the images into an images/ folder in your GitHub repository.

# Bank App Final Project — Private GitHub Runner and Terraform AWS Deployment

## Project Overview

This project focused on setting up a private self-hosted GitHub Actions runner on AWS EC2 and using it to deploy cloud infrastructure with Terraform.

The infrastructure was deployed into AWS using a CI/CD pipeline from GitHub Actions. The pipeline was executed through a private GitHub runner hosted on an EC2 instance. The deployment created core AWS resources such as VPC, subnets, route tables, internet gateway, NAT gateways, Elastic IP addresses, security groups, and an RDS MySQL database.

The project also included troubleshooting real CI/CD, Terraform, YAML, GitHub Actions, Namecheap DNS, and AWS deployment issues.

---

## Tools and Services Used

- AWS EC2
- AWS VPC
- AWS RDS
- AWS IAM
- AWS Elastic IP
- AWS NAT Gateway
- GitHub Actions
- Self-hosted GitHub Runner
- Docker
- Docker Compose
- Terraform
- tfsec
- VS Code
- Git and GitHub
- Namecheap DNS

---

## 1. Creating the Private GitHub Runner Server on AWS EC2

The first step was to create an EC2 instance that would act as the private GitHub runner.

In AWS, I opened the EC2 service and launched a new instance.

The instance was configured with the following details:

- Instance name: `git-runner-private`
- Operating system: Ubuntu
- Instance type: `t3.medium`
- Storage: 20 GB
- Key pair: Created for SSH/instance access
- Security group: Existing or newly created security group

After creating the instance, I connected to it using EC2 Instance Connect.

![EC2 Instance Connect Page](images/ec2-instance-connect.png)

![Connected Ubuntu EC2 Terminal](images/ec2-ubuntu-terminal.png)

---

## 2. Installing Docker and Docker Compose on the Runner Server

After connecting to the EC2 instance, I updated the server and installed Docker and Docker Compose.

```bash
sudo apt update -y
sudo apt upgrade -y
sudo apt install docker.io docker-compose -y

After installation, I confirmed Docker was installed by running:

sudo docker ps

3. Adding the Ubuntu User to the Docker Group

To allow the Ubuntu user to run Docker commands without permission issues, I added the user to the Docker group.

At first, an incorrect command was entered:

sudo docker -aG docker $USER

This produced an error because docker was used instead of usermod.

The correct command used was:

sudo usermod -aG docker $USER
sudo su - ubuntu

This allowed the user permissions to work properly with Docker.

4. Creating the Git Runner Project Directory

I created a working directory for the GitHub runner setup.

mkdir gitrunner
cd gitrunner

Inside this directory, I created three main configuration files:

nano .env
nano docker-compose.yaml
nano Dockerfile
nano entrypoint.sh

The files created were:

.env
docker-compose.yaml
Dockerfile
entrypoint.sh

I confirmed the files using:

ll

5. Creating the Environment File

The .env file was created to store the GitHub runner configuration values.

The file contained values such as:

GITHUB_PAT=********
REPO=https://github.com/username/repository-name
RUNNER_NAME=git-runner-private

Important: Secret values such as GitHub personal access tokens should not be exposed in screenshots or committed to GitHub.

The .env file was later added to .gitignore so that sensitive credentials would not be pushed to the repository.

6. Creating the Dockerfile for the GitHub Runner

The Dockerfile was created to build the GitHub Actions runner image.

The Dockerfile handled:

Installing dependencies
Downloading the latest GitHub Actions runner
Creating a non-root runner user
Setting runner permissions
Copying the entrypoint script
Starting the runner process

7. Creating the Entrypoint Script

The entrypoint.sh file was created to register the GitHub runner automatically.

The script requested a registration token from GitHub and used it to configure the self-hosted runner.

#!/bin/bash
set -e

cd /home/runner

echo "Fixing permissions for /home/runner/_work..."
mkdir -p /home/runner/_work
chown -R runner:runner /home/runner/_work

TOKEN_URL="https://api.github.com/repos/${REPO}/actions/runners/registration-token"

echo "Requesting registration token for $REPO..."
RUNNER_TOKEN=$(curl -s -X POST \
  -H "Authorization: token ${GITHUB_PAT}" \
  "${TOKEN_URL}" | jq -r .token)

echo "Registering runner: $RUNNER_NAME"
./config.sh --unattended \
  --url "https://github.com/${REPO}" \
  --token "${RUNNER_TOKEN}" \
  --name "${RUNNER_NAME}" \
  --labels self-hosted,docker,terraform \
  --work _work \
  --replace

echo "Starting runner...."
exec ./run.sh

8. Fixing Docker Compose Configuration

During the setup, there was an issue in the docker-compose.yaml file where the service was configured with:

user: root

This caused issues with the GitHub runner setup. The root user setting was removed from the Docker Compose configuration.

After fixing the file, the container was restarted:

docker-compose down
docker-compose up -d

9. Building and Starting the GitHub Runner Container

The runner container was started using Docker Compose:

docker-compose up -d

The image was built successfully and the GitHub runner container started.

docker ps
docker logs github-runner -f

At first, the runner registration failed with a 404 Not Found error. This was caused by an incorrect repository value in the .env file.

After correcting the repository format and runner configuration, the logs showed that the runner successfully connected to GitHub.

The successful runner log showed:

Runner successfully added
Settings Saved.
Connected to GitHub
Listening for Jobs
10. Verifying the Runner in GitHub

After the runner connected successfully, I verified it in GitHub.

Path:

GitHub Repository → Settings → Actions → Runners

The runner appeared as:

git-runner-private

Labels:

self-hosted
Linux
X64
docker
terraform

The runner showed as Idle, which means it was online and waiting for jobs.

Note: A runner only shows as active when it is currently executing a job. After a job finishes, it returns to idle. This is normal.

11. Creating GitHub Actions Workflow Folder

In VS Code, the workflow directory was created inside the repository:

.github/workflows

Inside the folder, the CI/CD workflow file was created:

cicd.yaml

This file was used to define the Terraform deployment pipeline.

12. Creating AWS IAM User for GitHub Actions

To allow GitHub Actions to deploy infrastructure into AWS, I created an IAM user.

Steps followed:

Opened AWS IAM
Created a new IAM user
Attached policies directly
Added AdministratorAccess for the project environment
Created the user
Opened the user security credentials
Created an access key for CLI usage

The following GitHub repository secrets were created:

AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
TF_API_TOKEN

13. Creating Terraform Cloud API Token

The workflow required a Terraform API token because the workflow used:

cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

To create the token:

Logged into Terraform Cloud / HCP Terraform
Created an organization
Created a workspace
Opened user settings
Went to Tokens
Created a new API token
Added the token to GitHub Secrets as TF_API_TOKEN

14. Creating the CI/CD Workflow

The GitHub Actions workflow was configured to:

Run on push to the main branch
Run on the private self-hosted runner
Configure AWS credentials
Set up Terraform
Clean old Terraform cache
Install tfsec
Run Terraform Init
Run Terraform Format
Run Terraform Validate
Run Terraform Security Scan
Run Terraform Plan
Run Terraform Apply

Final workflow structure:

name: Deploy to AWS with Terraform

on:
  workflow_dispatch:
  push:
    branches:
      - main

permissions:
  contents: write
  id-token: write

jobs:
  deploy-iac:
    runs-on: [self-hosted, docker, terraform]

    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: ${{ secrets.AWS_REGION }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Confirm private GitHub runner
        run: |
          echo "Running on private self-hosted GitHub runner"
          whoami
          docker --version

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ secrets.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

      - name: Clean old Terraform cache
        run: rm -rf .terraform .terraform.lock.hcl

      - name: Install tfsec for security scanning
        run: |
          curl -sLo tfsec https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64
          chmod +x tfsec
          ./tfsec --version

      - name: Terraform Init
        run: terraform init -reconfigure -input=false

      - name: Terraform Format
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Security Scan
        continue-on-error: true
        run: ./tfsec .

      - name: Terraform Plan
        run: terraform plan -input=false

      - name: Terraform Apply
        run: terraform apply -auto-approve -input=false
15. Git Workflow Used

After making changes in VS Code, I used Git commands to add, commit, and push changes to GitHub.

git status
git add .
git commit -m "Update Terraform variables and CI/CD workflow"
git push origin main

A warning appeared about line endings:

LF will be replaced by CRLF

This was a Git line-ending warning and did not stop the project from working.

Troubleshooting and Fixes
Issue 1: GitHub Runner Showing Idle

The GitHub runner showed Idle after registration.

This was not an error. It simply meant the runner was online and waiting for jobs.

The runner becomes active only when a workflow job is running.

Issue 2: Workflow Not Running Because No Workflow File Existed

Initially, the GitHub repository did not have a proper workflow file.

To fix this, I created:

.github/workflows/cicd.yaml

After pushing this file, GitHub Actions was able to detect and run the workflow.

Issue 3: Terraform Init Failed Due to S3 Backend Access

During one of the early workflow runs, Terraform Init failed with an S3 backend access error.

The error showed that Terraform could not access the remote state object in an S3 bucket.

The fix was to verify the S3 backend configuration and ensure the backend bucket existed and was accessible.

This was an important troubleshooting step because Terraform uses backend storage to manage state.

Issue 4: Helm Provider Version Error

Terraform Validate failed because the latest Helm provider version did not support the existing syntax used in the project.

The error involved blocks such as:

kubernetes {
}

and:

set {
}

The fix was to pin the Helm provider to version ~> 2.17.0.

In 01-provider.tf, the Helm provider was added to required_providers:

helm = {
  source  = "hashicorp/helm"
  version = "~> 2.17.0"
}

The Kubernetes provider was also added:

kubernetes = {
  source  = "hashicorp/kubernetes"
  version = "~> 2.37.1"
}

This allowed the existing Terraform syntax to work without rewriting the Helm resources.

Issue 5: Terraform Format Error

At one point, the workflow failed at the Terraform Format stage.

The error showed files that needed formatting:

01-provider.tf
main.tf
terraform.tfvars
variable.tf

The fix was to run:

terraform fmt -recursive

Then I committed and pushed the formatted files.

Issue 6: Terraform Security Scan Failed

The pipeline later reached the Terraform Security Scan stage.

The security scan detected multiple issues:

10 passed
23 potential problems detected
3 critical
6 high
9 medium
5 low

Some of the detected issues included:

EKS public cluster access enabled
Security group egress open to 0.0.0.0/0
RDS encryption not enabled
RDS backup retention too low
RDS IAM authentication not enabled
RDS deletion protection not enabled
ECR image tags are mutable
EKS secrets encryption not enabled
EKS control plane logging not enabled
Public subnets assigning public IPs
Security group rules without descriptions

Since this was a learning/final project environment, I configured the scan as an advisory step by adding:

continue-on-error: true

This allowed the workflow to continue while still recording the security findings for future improvement.

Issue 7: YAML Syntax Error

After adding continue-on-error, the workflow failed because of a YAML indentation error.

The error message showed:

Invalid workflow file
You have an error in your yaml syntax

The fix was to correctly align the YAML like this:

      - name: Terraform Security Scan
        continue-on-error: true
        run: ./tfsec .

After fixing the indentation and pushing again, the workflow ran successfully.

Issue 8: Terraform Apply Commented Out

The initial CI/CD workflow had the Terraform Apply stage commented out.

This meant the pipeline could test, validate, scan, and plan, but it would not actually create AWS resources.

The commented section was changed from:

# - name: Terraform Apply
#   run: terraform apply -auto-approve -input=false

to:

      - name: Terraform Apply
        run: terraform apply -auto-approve -input=false

A YAML indentation issue occurred while uncommenting this section, but it was corrected by aligning the Terraform Apply step with the Terraform Plan step.

Issue 9: Namecheap API Request Failed

During the full terraform apply, the deployment progressed very far and created several AWS and Kubernetes resources. However, the workflow failed when Terraform attempted to update DNS records through the Namecheap provider.

The error was:

Error: Invalid request IP: 18.144.224.57

The error pointed to:

module-dns/namecheap-name-servers.tf

Terraform was trying to update Namecheap DNS records through the Namecheap API, but Namecheap rejected the request IP.

Since DNS had already been configured manually in Namecheap, I disabled the Namecheap DNS automation module.

In main.tf, I commented out:

# module "namecheap-deployment" {
#   source = "./module-dns"
#   domain_name = var.domain_name
#   namecheap_api_user = var.namecheap_api_user
#   namecheap_api_key = var.namecheap_api_key
#   namecheap_username = var.namecheap_username
#   namecheap_client_ip = var.namecheap_client_ip
# }

In 01-provider.tf, I also commented out the active Namecheap provider block:

# provider "namecheap" {
#   user_name   = var.namecheap_username
#   api_user    = var.namecheap_api_user
#   api_key     = var.namecheap_api_key
#   client_ip   = var.namecheap_client_ip
#   use_sandbox = false
# }

Then I committed and pushed the fix:

terraform fmt -recursive
git status
git add main.tf 01-provider.tf
git commit -m "Disable Namecheap DNS automation"
git push origin main

After this fix, the deployment completed successfully.

Final Deployment Verification

After the successful GitHub Actions workflow, I verified the deployed resources in AWS.

The deployment was created in:

US East (N. Virginia) / us-east-1
RDS Database

Terraform deployed an RDS MySQL database.

Database identifier:

production-mysql-db

Status:

Available

Subnets

Terraform created 6 subnets, including public, private, and database subnets.

Route Tables

Terraform created 3 route tables:

production-private-route-table-1
production-private-route-table-2
production-public-route-table

Internet Gateway

Terraform created and attached an internet gateway:

production-internet-gateway

Elastic IP Addresses

Terraform created two Elastic IP addresses for the NAT gateways.

NAT Gateways

Terraform created two NAT gateways.

Status:

Available

Security Groups

Terraform created security groups for the environment, including:

Default security group
EKS cluster security group
Production MySQL security group

DNS and Domain Configuration

Route 53 was not used in this project because the domain was managed externally through Namecheap.

The original Terraform configuration attempted to automate DNS updates through the Namecheap provider, but Namecheap rejected the API request because of an IP restriction.

To resolve this, DNS automation was disabled in Terraform and DNS was handled manually from the Namecheap dashboard.

This means the infrastructure deployment remained successful while DNS configuration was separated from Terraform automation.

For a future production setup, the domain can be pointed manually to the AWS application endpoint, such as an AWS Load Balancer DNS name.

Final Outcome

By the end of this stage, the project achieved the following:

Created an EC2-based private self-hosted GitHub runner
Installed Docker and Docker Compose on the runner server
Built and registered the GitHub runner container
Connected the runner to GitHub Actions
Created GitHub repository secrets for AWS and Terraform
Created a CI/CD workflow using GitHub Actions
Configured Terraform Init, Format, Validate, Security Scan, Plan, and Apply
Fixed Terraform provider compatibility issues
Fixed YAML syntax and indentation errors
Handled tfsec security findings as advisory checks
Disabled Namecheap DNS automation after API/IP restriction
Successfully deployed AWS infrastructure in us-east-1
Verified deployed AWS resources in the AWS Console
Lessons Learned

This project helped me understand how private CI/CD runners work in real cloud environments. I learned how GitHub Actions can run through a self-hosted EC2 runner instead of GitHub-hosted runners, and how Terraform can be used to provision cloud infrastructure automatically.

I also learned how to troubleshoot real deployment errors, including:

Docker permission issues
GitHub runner registration errors
GitHub Actions workflow file errors
YAML indentation errors
Terraform provider version compatibility
Terraform backend access errors
Security scan findings
Namecheap API/IP restrictions
AWS infrastructure verification

The successful deployment confirmed that the private runner, GitHub Actions workflow, Terraform configuration, and AWS credentials were correctly connected.

Important Cost Note

Some AWS resources created in this project can generate charges, including:

NAT Gateways
RDS databases
EKS clusters
Load balancers
Elastic IP addresses

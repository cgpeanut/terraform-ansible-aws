```
Deploy terraform and ansible in AWS guide - roroxas

Step 1: Setup the Environment:
- Install terraform 
- Set Up AWS CLI and Ansible
- Set Up AWS IAM Permissions for Terraform

Step 3: Terraform Infrastructure as Code (IaaC)
- Understand terraform init, validate, plan and apply
- Persisting terraform state in S3 Back End
- Setting Up Multiple providers in Terraform
- Network Set Up part 1: Deploying VPCs, Internet GWs, and Subnets
- Network Set Up part 2: Deploying Multi-Region VPC Peering
- Network Set Up part 3: Deploying Security Groups
Hand-On -> Creating a multi-region network with VPC peering using SGs, IGW and RTs
- App VM Deployment part 1: Using Data Source (SSM Parameter Store) to fetch AMI IDs

#Get Linux AMI ID using SSM Parameter endpoint in us-east-1
data "aws_ssm_parameter" "linuxAmi" {
  provider = aws.region-master
  name     = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}

#Get Linux AMI ID using SSM Parameter endpoint in us-west-2
data "aws_ssm_parameter" "linuxAmiOregon" {
  provider = aws.region-worker
  name     = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}
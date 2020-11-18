```
Deploy terraform and ansible in AWS Guide - roroxas

Step 1: Setup the Environment:
- Install terraform in a control node unzip in /usr/local/bin

$ wget -c https://releases.hashicorp.com/terraform/0.12.28/terraform_0.12.28_linux_amd64.zip

- Set Up AWS CLI and Ansible (centos7)

$ sudo yum -y install python3-pip
$ pip3 install ansible --user
$ pip3 install awscli --user 
$ sudo yum -y install jq (optional)
$ mkdir $HOME/iac-deploy-tf-ansible && cd $HOME/iac-deploy-tf-ansible
$ wget https://github.com/cgpeanut/terraform-ansible-aws/blob/main/ansible.cfg
$ aws --version
$ ansible --version
$ aws configure (enter secret and access keys)
$ aws ec2 describe-instances

```
```
- Set Up AWS IAM Permissions for Terraform - will need permissions to create, update, and
  delete various AWS resources. We can do this in two ways 1. create a separate IAM user
  with the required permissions 2. create am EC2 (IAMrole) instance profile with required
  permissions and attach it to EC2. I'm choosing relaxed policy this time, just curious.

$ wget https://github.com/cgpeanut/terraform-ansible-aws/blob/main/data/relax_terraform_deployment_iam_policy.json
$ wget https://github.com/cgpeanut/terraform-ansible-aws/blob/main/data/strict_terraform_deployment_iam_policy.json

AWS Management Console -> IAM -> Policy -> Create Policy -> paste above -> Review -> Name &Descripttion: TerrafromUserPolicy -> Create policy

- attach the newly created terraform user policy to a IAM user or EC2 role.

    AWS Management Console -> IAM -> Users -> User Name: terraformuser -> Programmatic Access -> Attach existing Policy Search for: TerraformUserPolicy -> Tag Name:TFPolicy

```
```

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
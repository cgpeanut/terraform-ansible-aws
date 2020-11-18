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

Step 2: Set Up AWS IAM Permissions for Terraform - will need permissions to create, update, and delete various AWS resources. We can do this in two ways 1. create a separate IAM user with the required permissions 2. create am EC2 (IAMrole) instance profile with required permissions and attach it to EC2. I'm choosing relaxed policy this time, just curious.

$ wget https://github.com/cgpeanut/terraform-ansible-aws/blob/main/data/relax_terraform_deployment_iam_policy.json
$ wget https://github.com/cgpeanut/terraform-ansible-aws/blob/main/data/strict_terraform_deployment_iam_policy.json

AWS Management Console -> IAM -> Policy -> Create Policy -> paste above -> Review -> Name &Descripttion: TerrafromUserPolicy -> Create policy

- attach the newly created terraform user policy to a IAM user 1st method.

AWS Management Console -> IAM -> Users -> User Name: terraformuser -> Programmatic Access -> Attach existing Policy Search for: TerraformUserPolicy -> Tag Name:TFPolicy -> Create User -> Save the credentils securely.

2nd method create an EC2 role.

AWS Management Console -> Roles -> AWS service, select EC2 -> search for: TerraformUserPolicy -> Tag Name: RoleEC2TF -> Name: EC2TFRole Decription: Allows EC2 instances to call AWS services -> Create Role

```
```

Step 3: Terraform Infrastructure as Code (IaaC)

- Understand terraform init, validate, plan and apply

```
```
$ terraform init

1. Initializes working directory - downloads and includes all modules and providers (except third party) in Terraform file.
2. Needs to be run befor deploying infrastructure. - As other stages of Terraform deployment require provider, pluginsm and modules, this command needs to run first!
3. Syncs config, safe to run - configures backend for storing infrastructure state and does not modify or delete any existing configuration state.

$ terraform fmt (format)

1. Formats template for readability - Makes Terraform templates look stylish and readable.
2. Helps in keeping code consistent - Keeps the formatting consistent, expecially if teams are collaborating and tracking Terraform code through version control.
3. Safe to run at any time - Does not modify or add anything to code; only reformats it and rewrites it back to the files.

$ terraform validate

1. validate config files - Checks for syntax mistakes anbd internal consistency (typos and misconfigured resources)
2. Needs terraform init to be run first - Expects an initialized working directory, therefor the init command should be run before validate can be run.
3. Safe to run at any time - a use case would be ti run in order to check for issues in TF code before commiting to version control.

$ terraform plan

1. creates execution plan - calculates the delta between required state and current state to create an execution plan.
2. Fail-safe before actual deployment - A check if te dep[loyment execution plan matches expectation before creating or modifying any actual infrastructure.
3. Execution plan can be saved using the -out flag - warning sensitive configuration items would also be saved in a file as plaintext.

$ terraform apply

1. Deploy the execution plan!
2. By default will prompt before deploying
3. Will display execution plan again

```
```

Step 4: Persisting terraform state in S3 Back End

Terraform Backends:

- Determines how a state is stored.
- By default, state is stored on local disk.
- Variables cannot be used as input to Terraform block.

## Intructions on how to create an S3 bucket for persisting terraform backend. ##

1. login to terraform control node.

$ cd $HOME/iac-deploy-tf-ansible
$ aws s3api create-bucket --bucket terraformstatebucket041520
$ vim backend.tf

##### backend.tf #####

terraform {
  required_version = ">=0.12.0"
  required_providers {
    aws = ">=3.0.0"
  }
  backend "s3" {
    region  = "us-east-1"
    profile = "default"
    key     = "terraformstatefile" # note you created this in the console 
    bucket  = "terraformstatebucket041520"
  }
}

##### end of backend.tf #####

* Uploads the terraform state file to the AWS S3 bucket once terraform apply is invoked *
* Vital for code sharing! and the last state of the project is safely secured. *

$ terraform init
$ terraform fmt 

```
```

Step 5: Setting Up Multiple providers in Terraform

Terraform Providers - The source code for all terraform resources,  providers carry out interactions with vendor APIs such as AWS, GCP and Azure. They also provide logic for managing, updating and creating resources in Terraform.

The secret sauce behind defining multiple providers is using the parameter called "alias". This is how we can peg down a specific provider to an alias and then invoke the specific provider against a specific resource in our terraform code. Here's how: login control node

$ cd $HOME/iac-deploy-tf-ansible
$ vim variables.tf

# Creating terraform variables for two separate regions us-east-1 & us-east-2 #

```
```
##### Start of variables.tf #####

variable "profile" {
  type    = string
  default = "default"
}

variable "region-master" {
  type    = string
  default = "us-east-1"
}

variable "region-worker" {
  type    = string
  default = "us-west-2"
}

##### end of variables.tf #####
```
```

$ vim providers.tf

```
```
##### start of providers.tf #####

provider "aws" {
  profile = var.profile
  region  = var.region-master
  alias   = "region-master"
}

provider "aws" {
  profile = var.profile
  region  = var.region-worker
  alias   = "region-worker"
}

##### end of providers.tf #####
```
```

$ terraform init 

Note: checkout .terraform for state and plugins

# Step 6: Deploy Network Layout 
```

[<img src="https://github.com/cgpeanut/terraform-ansible-aws/blob/main/images/terraform_ansible_aws.pdf">]

```
Note: must have S3 bucket and multuple providers set. 


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
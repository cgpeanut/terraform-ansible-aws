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

1. 




```
```

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
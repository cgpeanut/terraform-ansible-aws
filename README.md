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

#### end of providers.tf #####

```
```

$ terraform init 

Note: checkout .terraform for state and plugins
```
Step 6: Deploy Network Layout 

[<img src="https://github.com/cgpeanut/terraform-ansible-aws/blob/main/images/deploy_network_layout.png">]
Note: must have S3 bucket and multuple providers must be already set via control node. 

# Network Set Up part 1: Deploying VPCs, Internet GWs, and Subnets

```
$ cd $HOME/iac-deploy-tf-ansible
$ cat backend.tf
$ cat providers.tf

$ vim networks.tf

```
```
##### Start of networks.tf #####

#Create VPC in us-east-1
resource "aws_vpc" "vpc_master" {
  provider             = aws.region-master # alias provideri.tf us-east-1
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "master-vpc-jenkins"
  }

}

#Create VPC in us-west-2
resource "aws_vpc" "vpc_master_oregon" {
  provider             = aws.region-worker
  cidr_block           = "192.168.0.0/16" # cidr must must have no overlap-vpc peeering 
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "worker-vpc-jenkins"
  }

}

#Create IGW in us-east-1
resource "aws_internet_gateway" "igw" {
  provider = aws.region-master
  vpc_id   = aws_vpc.vpc_master.id
}

#Create IGW in us-west-2
resource "aws_internet_gateway" "igw-oregon" {
  provider = aws.region-worker
  vpc_id   = aws_vpc.vpc_master_oregon.id
}

#Get all available AZ's in VPC for master region and pass it to datasource "azs"
data "aws_availability_zones" "azs" {
  provider = aws.region-master
  state    = "available"
}

#Create subnet # 1 in us-east-1
resource "aws_subnet" "subnet_1" {
  provider          = aws.region-master
  availability_zone = element(data.aws_availability_zones.azs.names, 0) # 1st element 
  vpc_id            = aws_vpc.vpc_master.id
  cidr_block        = "10.0.1.0/24"
}

#Note: the element function takes the list of availability zones by the above data sources
#call and picks out the 1st element in the lisyt which starts from 0 index [0].

#Create subnet #2  in us-east-1
resource "aws_subnet" "subnet_2" {
  provider          = aws.region-master
  vpc_id            = aws_vpc.vpc_master.id
  availability_zone = element(data.aws_availability_zones.azs.names, 1) # 2nd element
  cidr_block        = "10.0.2.0/24"
}


#Create subnet in us-west-2
resource "aws_subnet" "subnet_1_oregon" {
  provider   = aws.region-worker
  vpc_id     = aws_vpc.vpc_master_oregon.id
  cidr_block = "192.168.1.0/24"
}

##### end of networks.tf #####

```
# Network Set Up part 2: Deploying Multi-Region VPC Peering and setting up the routes so VPC can communicate over the VPC peering connection.

Objective: Deploy VPC Peering Connection

[<img src="https://github.com/cgpeanut/terraform-ansible-aws/blob/main/images/deploy_vpc_peering_connection.png">]

```

$ cd $HOME/iac-deploy-tf-ansible
$ vim networks.t

##### continue building from the previous networks.tf file ####

#Initiate Peering connection request from us-east-1
resource "aws_vpc_peering_connection" "useast1-uswest2" {
  provider    = aws.region-master
  peer_vpc_id = aws_vpc.vpc_master_oregon.id # spun-up vpc in uswest2
  vpc_id      = aws_vpc.vpc_master.id        # originating vpc useast1
  peer_region = var.region-worker 

}

#Accept VPC peering request in us-west-2 from us-east-1
resource "aws_vpc_peering_connection_accepter" "accept_peering" {
  provider                  = aws.region-worker
  vpc_peering_connection_id = aws_vpc_peering_connection.useast1-uswest2.id
  auto_accept               = true
}

#Create route table in us-east-1
resource "aws_route_table" "internet_route" {
  provider = aws.region-master
  vpc_id   = aws_vpc.vpc_master.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  route {
    cidr_block                = "192.168.1.0/24" # so incooming packets 192. to useast1
    vpc_peering_connection_id = aws_vpc_peering_connection.useast1-uswest2.id
  }
  lifecycle {
    ignore_changes = all
  }
  tags = {
    Name = "Master-Region-RT"
  }
}

#Overwrite default route table of VPC(Master) with our route table entries
resource "aws_main_route_table_association" "set-master-default-rt-assoc" {
  provider       = aws.region-master
  vpc_id         = aws_vpc.vpc_master.id
  route_table_id = aws_route_table.internet_route.id
}

#Create route table in us-west-2
resource "aws_route_table" "internet_route_oregon" {
  provider = aws.region-worker
  vpc_id   = aws_vpc.vpc_master_oregon.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw-oregon.id
  }
  route {
    cidr_block                = "10.0.1.0/24"
    vpc_peering_connection_id = aws_vpc_peering_connection.useast1-uswest2.id
  }
  lifecycle {
    ignore_changes = all
  }
  tags = {
    Name = "Worker-Region-RT"
  }
}

#Overwrite default route table of VPC(Worker) with our route table entries
resource "aws_main_route_table_association" "set-worker-default-rt-assoc" {
  provider       = aws.region-worker
  vpc_id         = aws_vpc.vpc_master_oregon.id
  route_table_id = aws_route_table.internet_route_oregon.id
}

##### continue building from the previous networks.tf file ####

```
To validate: AWS Management Console -> VPC -> useast1 region (nVa) -> Peering Connections

[<img src="https://github.com/cgpeanut/terraform-ansible-aws/blob/main/images/useast1-peering-connection.png">]

# Network Set Up part 3: Deploying Security Groups

[<img src="https://github.com/cgpeanut/terraform-ansible-aws/blob/main/images/deploy-security-groups.png">]

$ cd $HOME/iac-deploy-tf-ansible

$ vim security_groups.tf

```

##### start of security_groups.tf #####

#Create SG for LB, only TCP/80,TCP/443 and outbound access
resource "aws_security_group" "lb-sg" {
  provider    = aws.region-master
  name        = "lb-sg"
  description = "Allow 443 and traffic to Jenkins SG"
  vpc_id      = aws_vpc.vpc_master.id
  ingress {
    description = "Allow 443 from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "Allow 80 from anywhere for redirection"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1" # means all protocol
    cidr_blocks = ["0.0.0.0/0"]
  }
}

#security_group for our jenkins master
#Create SG for allowing TCP/8080 from * and TCP/22 from your IP in us-east-1
resource "aws_security_group" "jenkins-sg" {
  provider    = aws.region-master
  name        = "jenkins-sg"
  description = "Allow TCP/8080 & TCP/22"
  vpc_id      = aws_vpc.vpc_master.id
  ingress {
    description = "Allow 22 from our public IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.external_ip] # referenced in variables.tf file
  }
  ingress {
    description = "allow anyone on port 8080"
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    security_groups = [aws_security_group.lb-sg.id]
  }
  ingress {
    description = "allow traffic from us-west-2"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["192.168.1.0/24"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

#Create SG for allowing TCP/22 from your IP in us-west-2
resource "aws_security_group" "jenkins-sg-oregon" {
  provider = aws.region-worker

  name        = "jenkins-sg-oregon"
  description = "Allow TCP/8080 & TCP/22"
  vpc_id      = aws_vpc.vpc_master_oregon.id
  ingress {
    description = "Allow 22 from our public IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.external_ip]
  }
  ingress {
    description = "Allow traffic from us-east-1"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["10.0.1.0/24"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

##### end of security_groups.tf #####

```
$ vim variables.tf

Add these lines:
```

variable "external_ip" {
  type = string
  default = "0.0.0.0/0"
}

Y```

$ terraform fmt 
$ terraform validate
$ terraform plan
$ terraform apply --auto-approve

$ terraform destroy

```

Hand-On -> Creating a multi-region network with VPC peering using SGs, IGW and RTs
Objective:
  - Log in to the Terraform Controller Node EC2 Instance
  - Clone the GitHub Repo for Terraform Code
  - Deploy the Terraform Code

It can get cumbersome trying to track all the different routing components of a network,
especially in the fast-moving, dynamic IT operations world today. By maintaining your AWS
resources such as VPC, SGs, and IGWs using Terraform, you can track all of the changes as
code.

Let's go through creating a network setup complete with VPCs, subnets, security groups,
internet gateways, and VPC peering in AWS using Terraform. We are expected to have working
knowledge of VPC resources and basic network components within AWS.

```
[<img src="https://github.com/cgpeanut/terraform-ansible-aws/blob/main/images/vpc-peering.png">

login to EC2 instance control node

$ terraform --version

$ git clone https://github.com/cgpeanut/terraform-ansible-aws.git

$ cd hands-on-network-vpc-peering_code

$ ls 

network_setup.tf -> setup VPCs, multiple providers, cidr ranges for VPCs and subnets, definitions for security_groups which allows communications between various entities between the two VPCs

outputs.tf -> Outputs the VPC id of us-east-1 and us-west-2 and the peering connection ids between the two VPCs.

variables.tf -> passing external_ip by default accept all traffic (can specify your own public ip) defines the variables for the different regions

$ terraform init
$ terraform fmt
$ terraform validate
$ terraform plan
$ terrafomr apply

```
```
#####  begin of network_setup.tf #####

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

#Create VPC in us-east-1
resource "aws_vpc" "vpc_useast" {
  provider             = aws.region-master
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "master-vpc-jenkins"
  }

}

#Create VPC in us-west-2
resource "aws_vpc" "vpc_uswest" {
  provider             = aws.region-worker
  cidr_block           = "192.168.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "worker-vpc-jenkins"
  }

}

#Initiate Peering connection request from us-east-1
resource "aws_vpc_peering_connection" "useast1-uswest-2" {
  provider    = aws.region-master
  peer_vpc_id = aws_vpc.vpc_uswest.id
  vpc_id      = aws_vpc.vpc_useast.id
  #auto_accept = true
  peer_region = var.region-worker

}

#Create IGW in us-east-1
resource "aws_internet_gateway" "igw" {
  provider = aws.region-master
  vpc_id   = aws_vpc.vpc_useast.id
}

#Create IGW in us-west-2
resource "aws_internet_gateway" "igw-oregon" {
  provider = aws.region-worker
  vpc_id   = aws_vpc.vpc_uswest.id
}

#Accept VPC peering request in us-west-2 from us-east-1
resource "aws_vpc_peering_connection_accepter" "accept_peering" {
  provider                  = aws.region-worker
  vpc_peering_connection_id = aws_vpc_peering_connection.useast1-uswest-2.id
  auto_accept               = true
}

#Create route table in us-east-1
resource "aws_route_table" "internet_route" {
  provider = aws.region-master
  vpc_id   = aws_vpc.vpc_useast.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  route {
    cidr_block                = "192.168.1.0/24"
    vpc_peering_connection_id = aws_vpc_peering_connection.useast1-uswest-2.id
  }
  lifecycle {
    ignore_changes = all
  }
  tags = {
    Name = "Master-Region-RT"
  }
}

#Overwrite default route table of VPC(Master) with our route table entries
resource "aws_main_route_table_association" "set-master-default-rt-assoc" {
  provider       = aws.region-master
  vpc_id         = aws_vpc.vpc_useast.id
  route_table_id = aws_route_table.internet_route.id
}
#Get all available AZ's in VPC for master region
data "aws_availability_zones" "azs" {
  provider = aws.region-master
  state    = "available"
}

#Create subnet # 1 in us-east-1
resource "aws_subnet" "subnet_1" {
  provider          = aws.region-master
  availability_zone = element(data.aws_availability_zones.azs.names, 0)
  vpc_id            = aws_vpc.vpc_useast.id
  cidr_block        = "10.0.1.0/24"
}

#Create subnet #2  in us-east-1
resource "aws_subnet" "subnet_2" {
  provider          = aws.region-master
  vpc_id            = aws_vpc.vpc_useast.id
  availability_zone = element(data.aws_availability_zones.azs.names, 1)
  cidr_block        = "10.0.2.0/24"
}


#Create subnet in us-west-2
resource "aws_subnet" "subnet_1_oregon" {
  provider   = aws.region-worker
  vpc_id     = aws_vpc.vpc_uswest.id
  cidr_block = "192.168.1.0/24"
}

#Create route table in us-west-2
resource "aws_route_table" "internet_route_oregon" {
  provider = aws.region-worker
  vpc_id   = aws_vpc.vpc_uswest.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw-oregon.id
  }
  route {
    cidr_block                = "10.0.1.0/24"
    vpc_peering_connection_id = aws_vpc_peering_connection.useast1-uswest-2.id
  }
  lifecycle {
    ignore_changes = all
  }
  tags = {
    Name = "Worker-Region-RT"
  }
}

#Overwrite default route table of VPC(Worker) with our route table entries
resource "aws_main_route_table_association" "set-worker-default-rt-assoc" {
  provider       = aws.region-worker
  vpc_id         = aws_vpc.vpc_uswest.id
  route_table_id = aws_route_table.internet_route_oregon.id
}


#Create SG for allowing TCP/8080 from * and TCP/22 from your IP in us-east-1
resource "aws_security_group" "jenkins-sg" {
  provider    = aws.region-master
  name        = "jenkins-sg"
  description = "Allow TCP/8080 & TCP/22"
  vpc_id      = aws_vpc.vpc_useast.id
  ingress {
    description = "Allow 22 from our public IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.external_ip]
  }
  ingress {
    description = "allow anyone on port 8080"
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "allow traffic from us-west-2"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["192.168.1.0/24"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

#Create SG for LB, only TCP/80,TCP/443 and access to jenkins-sg
resource "aws_security_group" "lb-sg" {
  provider    = aws.region-master
  name        = "lb-sg"
  description = "Allow 443 and traffic to Jenkins SG"
  vpc_id      = aws_vpc.vpc_useast.id
  ingress {
    description = "Allow 443 from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "Allow 80 from anywhere for redirection"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description     = "Allow traffic to jenkins-sg"
    from_port       = 0
    to_port         = 0
    protocol        = "tcp"
    security_groups = [aws_security_group.jenkins-sg.id]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

#Create SG for allowing TCP/22 from your IP in us-west-2
resource "aws_security_group" "jenkins-sg-oregon" {
  provider = aws.region-worker

  name        = "jenkins-sg-oregon"
  description = "Allow TCP/8080 & TCP/22"
  vpc_id      = aws_vpc.vpc_uswest.id
  ingress {
    description = "Allow 22 from our public IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.external_ip]
  }
  ingress {
    description = "Allow traffic from us-east-1"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["10.0.1.0/24"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

##### end of network_setup.tf #####

```
```

##### start of outputs.tf #####

output "VPC-ID-US-EAST-1" {
  value = aws_vpc.vpc_useast.id
}

output "VPC-ID-US-WEST-2" {
  value = aws_vpc.vpc_uswest.id
}

output "PEERING-CONNECTION-ID" {
  value = aws_vpc_peering_connection.useast1-uswest-2.id
}

##### end of variables.tf #####

```
```

##### start of variables.tf #####

variable "external_ip" {
  type    = string
  default = "0.0.0.0/0"
}

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

To Validate:

AWS management Console -> search for VPC -> VPC - master-vpc-jenkins is theone we soun up
check the route: click Description Tab -> Click Route Table (bottom) -> Route






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
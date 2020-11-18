Implementing terraform and ansible in AWS

```
Create a Control node in AWS  and install terraform
    1. Download terraform binary => 0.12.x
      $ wget -c https://releases.hashicorp.com/terraform/0.12.28/terraform_0.12.28_linux_amd64.zip
    2. Python3 & PIP needs to be installed on all nodes
      $ yum -y install python3-pip
    3. Install ansible (install via pip)
      $ pip3 install ansible --user
    4. Install AWS CLI (install via pip) 
      $ pip3 install awscli --user 
    5. jq (install via package manager) - OPTIONAL 
      $ yum -y install j
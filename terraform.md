# terraform command ref

## installation & setup

```bash
# install via tfenv (recommended — manages multiple versions)
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
tfenv install latest
tfenv use latest
terraform version

# or via package manager (macOS)
brew install terraform

# shell completion
terraform -install-autocomplete
```

## workspace & init

```bash
terraform init                      # initialize working dir, download providers
terraform init -upgrade             # upgrade providers to latest allowed version
terraform init -reconfigure         # reinitialize, ignore existing state
terraform init -backend=false       # skip backend init
terraform workspace list            # list workspaces
terraform workspace new <n>      # create workspace
terraform workspace select <n>   # switch workspace
terraform workspace show            # current workspace
terraform workspace delete <n>   # delete workspace
```

## plan & apply

```bash
terraform plan                      # show execution plan
terraform plan -out=tfplan          # save plan to file
terraform plan -var="key=value"     # pass variable inline
terraform plan -var-file="prod.tfvars"  # use var file
terraform plan -target=resource.name    # plan specific resource
terraform plan -destroy             # plan a destroy
terraform apply                     # apply changes (prompts for confirmation)
terraform apply -auto-approve       # apply without confirmation
terraform apply tfplan              # apply saved plan file
terraform apply -var="key=value"    # apply with inline var
terraform apply -target=resource.name   # apply specific resource
terraform apply -replace=resource.name  # force replace a resource
terraform destroy                   # destroy all managed resources
terraform destroy -auto-approve     # destroy without confirmation
terraform destroy -target=resource.name # destroy specific resource
```

## state

```bash
terraform state list                # list all resources in state
terraform state show <resource>     # show state for a resource
terraform state mv <src> <dst>      # move/rename resource in state
terraform state rm <resource>       # remove resource from state (stop managing)
terraform state pull                # output current state as JSON
terraform state push <file>         # push a state file
terraform import <resource> <id>    # import existing infra into state
terraform refresh                   # sync state with real infrastructure
```

## output & inspect

```bash
terraform output                    # show all outputs
terraform output <name>             # show specific output
terraform output -json              # outputs as JSON
terraform show                      # human-readable state or plan
terraform show -json tfplan         # plan as JSON
terraform graph                     # generate dependency graph (dot format)
terraform graph | dot -Tsvg > graph.svg  # render as SVG
```

## validate & format

```bash
terraform validate                  # validate config syntax and logic
terraform fmt                       # format all .tf files in place
terraform fmt -check                # check formatting (exit 1 if unformatted)
terraform fmt -diff                 # show formatting diff
terraform fmt -recursive            # format subdirectories too
```

## providers

```bash
terraform providers                 # show required providers
terraform providers lock            # update dependency lock file
terraform providers mirror <dir>    # download providers to local dir
```

## console & debug

```bash
terraform console                   # interactive expression evaluator
> var.region                        # inspect a variable
> local.name                        # inspect a local
> aws_instance.web.id               # inspect a resource attribute
> cidrsubnet("10.0.0.0/16", 8, 1)   # test functions

TF_LOG=DEBUG terraform apply        # verbose logging
TF_LOG=TRACE terraform plan         # maximum verbosity
TF_LOG_PATH=./tf.log terraform plan # log to file
```

## file structure (recommended layout)

```
project/
├── main.tf           # primary resources
├── variables.tf      # input variable declarations
├── outputs.tf        # output declarations
├── locals.tf         # local value definitions
├── providers.tf      # provider configuration
├── versions.tf       # required_providers + terraform version
├── terraform.tfvars  # variable values (do not commit secrets)
├── .terraform.lock.hcl  # provider lock file (commit this)
└── modules/
    └── mymodule/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

## configuration syntax

### aws provider

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket = "my-tf-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.region
}
```

### vultr provider

```hcl
terraform {
  required_providers {
    vultr = {
      source  = "vultr/vultr"
      version = "~> 2.0"
    }
  }
}

# API key via environment variable (recommended — never hardcode)
# export VULTR_API_KEY="your_api_key_here"

provider "vultr" {
  api_key     = var.vultr_api_key   # or omit and use VULTR_API_KEY env var
  rate_limit  = 100                 # ms between API calls (default: 500)
  retry_limit = 3
}

# common resources
resource "vultr_instance" "web" {
  plan      = "vc2-1c-1gb"          # 1 vCPU, 1GB RAM
  region    = "lax"                  # Los Angeles
  os_id     = 1743                   # Ubuntu 22.04 LTS (check: terraform plan)
  label     = "web-server"
  hostname  = "web-01"
  tag       = "production"

  ssh_key_ids = [vultr_ssh_key.default.id]

  user_data = file("${path.module}/cloud-init.yaml")
}

resource "vultr_ssh_key" "default" {
  name    = "default"
  ssh_key = file("~/.ssh/id_ed25519.pub")
}

resource "vultr_firewall_group" "default" {
  description = "default firewall"
}

resource "vultr_firewall_rule" "ssh" {
  firewall_group_id = vultr_firewall_group.default.id
  protocol          = "tcp"
  ip_type           = "v4"
  subnet            = "0.0.0.0"
  subnet_size       = 0
  port              = "22"
  notes             = "allow ssh"
}

# look up available plans and regions
data "vultr_plan" "starter" {
  filter {
    name   = "vcpu_count"
    values = ["1"]
  }
}

data "vultr_region" "lax" {
  filter {
    name   = "id"
    values = ["lax"]
  }
}

variable "vultr_api_key" {
  type      = string
  sensitive = true
  default   = ""   # leave blank — set via VULTR_API_KEY env var or .tfvars
}
```

### variables

```hcl
variable "region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "tags" {
  type = map(string)
  default = {
    env = "prod"
  }
}

variable "db_password" {
  type      = string
  sensitive = true           # masked in logs and output
}
```

### locals

```hcl
locals {
  name_prefix = "${var.env}-${var.project}"
  common_tags = {
    env     = var.env
    project = var.project
    owner   = "ak"
  }
}
```

### outputs

```hcl
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of web instance"
}

output "db_password" {
  value     = var.db_password
  sensitive = true
}
```

### resources & data sources

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  tags          = local.common_tags

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [tags]
  }
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-24.04-amd64-server-*"]
  }
}
```

### meta-arguments

```hcl
# count
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = var.ami
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }
}

# for_each (preferred over count for maps/sets)
resource "aws_s3_bucket" "this" {
  for_each = toset(["logs", "backups", "assets"])
  bucket   = "${local.name_prefix}-${each.key}"
}

# depends_on
resource "aws_instance" "app" {
  depends_on = [aws_db_instance.db]
}
```

### expressions & functions

```hcl
# conditionals
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"

# string interpolation
name = "${var.project}-${var.env}"

# common functions
length(var.list)
toset(var.list)
tomap({a = 1, b = 2})
merge(local.common_tags, { extra = "tag" })
lookup(var.map, "key", "default")
element(var.list, 0)
flatten([[1,2],[3,4]])
join(",", var.list)
split(",", var.string)
format("%-10s %s", "hello", "world")
cidrsubnet("10.0.0.0/16", 8, 1)
jsonencode({ key = "value" })
jsondecode(file("config.json"))
file("${path.module}/script.sh")
templatefile("${path.module}/tpl.tftpl", { name = var.name })
```

## modules

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = local.name_prefix
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}

# local module
module "myapp" {
  source = "./modules/myapp"
  env    = var.env
}
```

## useful patterns

```bash
# plan and apply in one step (CI-safe)
terraform plan -out=tfplan && terraform apply tfplan

# target a single resource for quick iteration
terraform apply -target=aws_instance.web

# check what would be destroyed before running destroy
terraform plan -destroy

# import an existing resource
terraform import aws_s3_bucket.logs my-existing-bucket-name

# unlock state if locked (use with caution)
terraform force-unlock <lock-id>

# list workspaces and switch
terraform workspace list
terraform workspace select prod
```

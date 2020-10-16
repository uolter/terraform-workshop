# Terraform & Terragrunt
## 16 Oct 2020 Workshop

<!-- .slide: data-transition="zoom" -->

---

## Terraform

> Infrastructure as Code to provision and manage any cloud, infrastructure, or service

---
## Key features 

- Incrastrucure as a code
- Resource graph
- Execution plan

---
## Infrastructure as a code

>  define infrastructure in a simple, human-readable configuration language called **HCL** (HashiCorp Configuration Language)  

---
## Providers

* which plugin to use to access resources
* a pluging calls the api
* there are many [providers](https://registry.terraform.io/browse/providers?tier=partner)

---

### azure provider example

```
provider "azurerm" {
    version = "~>2.0"
    features {}
}


resource "azurerm_resource_group" "myterraformgroup" {
    name     = "myResourceGroup"
    location = "eastus"

    tags = {
        environment = "Terraform Demo"
        created_by  = "terraform"
    }
}

```

---

### AWS provider example

```
provider "aws" {
  profile    = "default"
  region     = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/24"
}
```

---
## Initialize a project

```
> terraform init
```

* pull down the provider(s)

* initialize the state file

---

## Workflow

- Individual
    
    write ->  plan (& check) -> create

- Team

  checkout latest code -> write & plan a PR --> create & merge
  
  
---

## Main Commands

```
> # format the code 
> terraform fmt [-recursive]

> # validate the code
> terraform validate 

> # shows changes
> terraform plan

> # apply changes
> terraform apply

> # destroy resources
> terraform destroy 

> # import an existing resource
> terraform import <resource> <resource id>
```

---
## Dependency (Implicit)

<div align="left">_main.tf_</div>
```
resource "azurerm_resource_group" "mygroup" {
    name     = "myResourceGroup"
    location = "eastus"

    tags = {
        environment = "Demo"
    }
}

resource "azurerm_virtual_network" "mynetwork" {
    name                = "myVnet"
    address_space       = ["10.0.0.0/16"]
    location            = "eastus"
    resource_group_name = azurerm_resource_group.mygroup.name

    tags = {
        environment = "Demo"
    }
}

resource "azurerm_subnet" "mysubnet" {
    name                 = "mySubnet"
    resource_group_name  = azurerm_resource_group.mygroup.name
    virtual_network_name = azurerm_virtual_network.mynetwork.name
    address_prefixes       = ["10.0.2.0/24"]
}
```

---

## Dependency (explicit)

<div align="left">_main.tf_</div>

```
... 
resource "azurerm_virtual_machine" "web-linux-vm" {
  
  location = azurerm_resource_group.network-rg.location
  resource_group_name   = azurerm_resource_group.network-rg.name
  name = "linux-${random_string.random-linux-vm.result}-vm"
  ....
  
  
  os_profile {
    computer_name = "linux-${random_string.random-linux-vm.result}-vm"
    admin_username = var.web-linux-admin-username
    admin_password = random_password.web-linux-vm-password.result
    custom_data    = file("azure-user-data.sh")
  }
  
  os_profile_linux_config {
    disable_password_authentication = false
  }
  
  depends_on=[azurerm_storage_account.example]
  
  tags = {
    environment = "Demo"
  }
}

resource "azurerm_storage_account" "example" {
  name                     = "storageaccountname"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  tags = {
    environment = "Demo"
  }
}
```

---

## State management


* _terraform.tfstate_ is the **most importart file**
* it **tracks** resources that have been created
* **local**: when we run the plan command for the first time the state file is created.
* **remote**: state file is stored somewhere else
    * S3 bucket
    * storage account

---
## Backend

```
terraform {
  backend "azurerm" {
    resource_group_name  = "StorageAccount-ResourceGroup"
    storage_account_name = "abcd1234"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```
---

## Backend security

> secrets are stored in the state file.

Keep the state file **encrypted** at rest.

---

## Advanced backend


```
>$ # download state file
>$ terraform state pull > terraform.tfstate
>$ # edit the file
>$ terraform push terraform.tfstate
```
- Be very carfeful
- Always make a backup copy


---

## Variables

Input parameters allowing aspects of the module to be customized without altering the module's own source code, and allowing modules to be shared between different configurations.

```
variable "vpc_cidr" {
    type        = string
    description = "the vpc cidr range" # optional
    default     = "10.0.0.0/24" # optional
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}

```

---

## [Types](https://github.com/pagopa/io-infrastructure-modules-new/blob/master/azurerm_app_service/vars.tf)

* string
* number
* bool
* list(< type >)
* map
* tuple([<type>, < type >, < type >])
* object({})


---

## Assign Variables

<br/>

```
export TF_VAR_vpc_cidr="10.10.0.0/16"
```


```
terraform plan -var="vpc_cidr=10.1.10.0/19" -var="name=main"
```

<div align="left">_terraform.tfvars_</div>

```
vpc_cird = "10.0.0.0/32"
name     =  "myvpc"
```

```
terraform plan -var-file=./dev.tfvars
terraform plan -var-file=./prod.tfvars
```

---

## Order values loading

* environment variables
* terraform.tfvars
* terraform.tfvars.json
* any -var-file command line option

---

## [Modules](https://github.com/pagopa/io-infrastructure-modules-new/blob/master/azurerm_function_app/main.tf)

* block of reusable code
* any folder with terraform code in there
* [registry.terraform.io](https://registry.terraform.io/browse/modules) 

```
├── main.tf
|
└── vnet
    ├── main.tf
    ├── outputs.tf
    └── variables.tf

```

---

## Loop with count 

<div align="left">main.tf</div>

```
provider "aws" {
  region = "us-east-1"
}

resource "aws_iam_user" "users" {
  count = length(var.user_names)
  name  = var.user_names[count.index]
}
```

<div align="left">variables.tf</div>

```
variable "user_names" {
  description = "Create IAM users with these names"
  type        = list(string)
  default     = ["Paul_Dirac", "Erwin_Schrodinger", 
                 "Wolfgang_Pauli"]
}
```
---

## [Dynamic blocks](https://github.com/pagopa/io-infrastructure-modules-new/blob/1af3569706d12c33aea1a540896a47db0e38362b/azurerm_function_app/main.tf#L83)

```
    dynamic "ip_restriction" {
      for_each = var.allowed_subnets
      iterator = subnet

      content {
        subnet_id = subnet.value
      }
    }
  }
```

---

## Terragrunt

* keep your terraform code dry
* keep your terraform state configuration dry
* keep your CLI flags dry
* exacute terraform commands on multiple modules at once

<!-- .slide: data-transition="zoom" -->

---


## [Terraform dry code](https://github.com/pagopa/io-infrastructure-live-new/blob/master/prod/westeurope/common/resource_group/terragrunt.hcl)

```
# Include all settings from the root terragrunt.hcl file
include {
  path = find_in_parent_folders()
}

terraform {
  source = "git::git@github.com:pagopa/io-infrastructure-modules-new.git//azurerm_resource_group?ref=v2.1.0"
}

inputs = {
  name = "common"
}
```

---
## Terreform dry code

* manage many environment without duplications

```
└── live
    ├── prod
    │   ├── app
    │   │   └── main.tf
    │   ├── mysql
    │   │   └── main.tf
    │   └── vpc
    │       └── main.tf
    ├── qa
    │   ├── app
    │   │   └── main.tf
    │   ├── mysql
    │   │   └── main.tf
    │   └── vpc
    │       └── main.tf
    └── stage
        ├── app
        │   └── main.tf
        ├── mysql
        │   └── main.tf
        └── vpc
            └── main.tf
```

---

## Terraform dry code

```
└── live
    ├── prod
    │   ├── app
    │   │   └── terragrunt.hcl
    │   ├── mysql
    │   │   └── terragrunt.hcl
    │   └── vpc
    │       └── terragrunt.hcl
    ├── qa
    │   ├── app
    │   │   └── terragrunt.hcl
    │   ├── mysql
    │   │   └── terragrunt.hcl
    │   └── vpc
    │       └── terragrunt.hcl
    └── stage
        ├── app
        │   └── terragrunt.hcl
        ├── mysql
        │   └── terragrunt.hcl
        └── vpc
            └── terragrunt.hcl
```

---

## [Dry backend configuration](https://github.com/pagopa/io-infrastructure-live-new/blob/50772ff78e5f92816e426bc5c579fb97ce47dc29/terragrunt.hcl#L13)

* backend configuration **does not** support variables or expression.

```
terraform {
  backend "azurerm" {
    resource_group_name  = "StorageAccount-ResourceGroup"
    storage_account_name = "abcd1234"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

---

## Commands on multiple modules

```
common
├── adgroup
│   └── terragrunt.hcl
├── application_insights
│   └── terragrunt.hcl
├── cdn
│   ├── cdn_endpoint_assets
│   │   └── terragrunt.hcl
.
.

├── dns_caa_record
│   └── terragrunt.hcl
├── dns_zone
│   └── terragrunt.hcl
.
├── notification_hub
│   └── terragrunt.hcl
.           └── terragrunt.hcl
├── redis
│   ├── redis_cache
│   │   └── terragrunt.hcl
│   ├── storage_accountte
│   │   └── terragrunt.hcl
│   └── subnet
│       └── terragrunt.hcl
```

---

## Commands on multiple modules
<br>
```
~$ terragrunt plan-all

~$ terragrunt apply-all

~$ terragrunt output-all

~$ terragrunt destroy-all
```

---

## IO Infrastructure

* [live](https://github.com/pagopa/io-infrastructure-live-new) 
* [modules](https://github.com/pagopa/io-infrastructure-modules-new) 

---

## Useful links

* [terraform](https://www.terraform.io/)
* [terragrunt](https://terragrunt.gruntwork.io/)
* [registry](https://registry.terraform.io/)
* [tfenv](https://github.com/cloudposse/tfenv)
* [tgenv](https://github.com/cunymatthieu/tgenv)
* [azure provider](https://www.terraform.io/docs/providers/index.html)
* [builtin function](https://www.terraform.io/docs/configuration/functions.html)


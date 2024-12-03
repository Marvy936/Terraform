# Terraform Training Notes

## Table of Contents
1. [Introduction to Terraform Commands](#introduction-to-terraform-commands)
    - [Initialize Terraform](#initialize-terraform)
    - [Plan Terraform Execution](#plan-terraform-execution)
    - [Apply Terraform Plan](#apply-terraform-plan)
    - [Destroy Terraform Managed Infrastructure](#destroy-terraform-managed-infrastructure)
2. [Useful Links](#useful-links)
3. [Setting Up a Tenant](#setting-up-a-tenant)
4. [Exporting Proxy Variables](#exporting-proxy-variables)
5. [Creating a Keypair](#creating-a-keypair)
6. [Creating Security Groups](#creating-security-groups)
7. [Adding Security Group Rules](#adding-security-group-rules)
8. [Creating and Managing an Instance](#creating-and-managing-an-instance)
9. [Variables in Terraform](#variables-in-terraform)
    - [Input Variables](#input-variables)
        - [Declaring Variables](#declaring-variables)
        - [Using Input Variables](#using-input-variables)
    - [Output Variables](#output-variables)
    - [Local Values](#local-values)
        - [Declaring Local Values](#declaring-local-values)
        - [Using Local Values](#using-local-values)
        - [Ephemeral Local Values](#ephemeral-local-values)
10. [Best Practices](#best-practices-for-variables-and-local-values)

## Introduction to Terraform Commands

1. **Initialize Terraform**
   ```bash
   terraform init
   ```
   Initializes a new or existing Terraform configuration by preparing the working directory and downloading provider plugins.

2. **Plan Terraform Execution**
   ```bash
   terraform plan
   ```
   Creates an execution plan, showing what actions Terraform will take to achieve the desired state defined in your configuration.

3. **Apply Terraform Plan**
   ```bash
   terraform apply
   ```
   Executes the actions proposed in the plan to make your infrastructure match the desired state.

4. **Destroy Terraform Managed Infrastructure**
   ```bash
   terraform destroy
   ```
   Removes the resources created by Terraform.

## Useful Links

- [OpenTelekomCloud Provider Documentation](https://registry.terraform.io/providers/opentelekomcloud/opentelekomcloud/latest/docs)
- [Terraform Release Downloads](https://releases.hashicorp.com/)

## Setting Up a Tenant

1. Create a new directory:
   ```bash
   mkdir 00-tenant-
   ```
2. Create a `provider.tf` file with the following configuration:

   ```hcl
   terraform {
       required_version = ">= 0.14.0"
       required_providers {
           openstack = {
               source = "terraform-provider-openstack/openstack"
               version = "~> 1.48.0"
           }
       }
   }

   provider "openstack" {
       user_name        = "devops-terraform"
       password         = "2eir6shohHpeing3Mee]do"
       user_domain_name = "OTC-EU-DE-00000000001000035578"
       tenant_id        = "54026885c74446c2b833f4dc7cb77bd2"
       auth_url         = "https://iam.eu-de.otc.t-systems.com:443/v3"
       region           = "eu-de"
   }
   ```

3. Initialize Terraform:
   ```bash
   terraform init
   ```

## Exporting Proxy Variables (if needed)
```bash
export https_proxy=http://10.14.38.3:3128
export http_proxy=http://10.14.38.3:3128
```

## Creating a Keypair

1. Create a `keypair.tf` file with the following content:
   ```hcl
   resource "openstack_compute_keypair_v2" "vyhonsky-keypair" {
       name       = "vyhonsky-keypair"
       public_key = "{public_key}"
   }
   ```

2. Save the execution plan to a file:
   ```bash
   terraform plan -out=plan.tf
   ```

3. Apply the saved plan:
   ```bash
   terraform apply "plan.tf"
   ```

4. List Terraform-managed resources:
   ```bash
   terraform state list
   ```

## Creating Security Groups

1. Create a `security-group.tf` file:
   ```hcl
   resource "openstack_networking_secgroup_v2" "sg-vyhonsky" {
       name               = "sg-vyhonsky"
       description        = "ALL"
       delete_default_rules = true
   }
   ```

2. Save and apply the security group plan:
   ```bash
   terraform plan -out=scplan
   terraform apply "scplan"
   ```

## Adding Security Group Rules

1. Create a `sg-rule.tf` file:
   ```hcl
   resource "openstack_networking_secgroup_rule_v2" "allow_ssh_vyhonsky" {
       direction         = "ingress"
       ethertype         = "IPv4"
       protocol          = "tcp"
       port_range_min    = 22
       port_range_max    = 22
       remote_ip_prefix  = "10.0.0.0/8"
       security_group_id = openstack_networking_secgroup_v2.sg-vyhonsky.id
   }

   resource "openstack_networking_secgroup_rule_v2" "outbound_ssh_vyhonsky" {
       direction         = "egress"
       ethertype         = "IPv4"
       protocol          = "tcp"
       remote_ip_prefix  = "10.0.0.0/8"
       security_group_id = openstack_networking_secgroup_v2.sg-vyhonsky.id
   }
   ```

2. Apply the rules:
   ```bash
   terraform apply "scplan"
   ```

## Creating and Managing an Instance

1. Create a new directory:
   ```bash
   mkdir 01-single-instance
   cd 01-single-instance
   ```

2. Create an `instance.tf` file:
   ```hcl
   data "openstack_networking_secgroup_v2" "sg-vyhonsky" {
       name = "sg-vyhonsky"
   }

   data "openstack_compute_keypair_v2" "vyhonsky-keypair" {
       name = "vyhonsky-keypair"
   }

   resource "openstack_compute_instance_v2" "vyhonsky-vm" {
       name          = "vyhonsky-vm"
       image_name    = "Standard_Ubuntu_22.04_latest"
       flavor_name   = "s3.medium.2"
       key_pair      = data.openstack_compute_keypair_v2.vyhonsky-keypair.name
       user_data     = file("mount_VM.sh")

       network {
           port = openstack_networking_port_v2.primary_port.id
       }
   }
   ```

3. Create a volume and attach it to the instance:
   ```hcl
   resource "openstack_blockstorage_volume_v3" "data0-vyhonsky" {
       name               = "data0-vyhonsky"
       size               = 10
       enable_online_resize = true
       volume_type        = "SSD"
   }

   resource "openstack_compute_volume_attach_v2" "data0-vyhonsky" {
       instance_id = openstack_compute_instance_v2.vyhonsky-vm.id
       volume_id   = openstack_blockstorage_volume_v3.data0-vyhonsky.id
   }
   ```

4. Include a shell script `mount_VM.sh` for configuring the volume:
   ```bash
   #!/bin/bash

   mkfs -t ext4 /dev/vdb
   mkdir -p /mnt/data
   mount /dev/vdb /mnt/data
   echo /dev/vdb /mnt/data ext4 defaults,nofail 0 2 >> /etc/fstab
   ```
After resizing volume, you have to resize mounted disk.
   ```bash
   resize2fs /dev/vdb
   ```
   
## Variables in Terraform

Variables in Terraform enable dynamic and reusable configurations. They are declared in `variables.tf` files or directly within your Terraform manifests.

### Input Variables

Input variables are values that you pass into Terraform configurations.

#### Declaring Variables

Variables must first be defined in a block or `variables.tf` file:
```hcl
variable "private_key_path" {
  type    = string
  default = "/home/user/.ssh/terraform_rsa"
}

variable "public_key" {
  type    = string
  default = "ssh-rsa terraform_public_key"
}

variable "zones" {
  type = map
  default = {
    "amsterdam" = "nl-ams1"
    "london"    = "uk-lon1"
    "frankfurt" = "de-fra1"
    "helsinki1" = "fi-hel1"
    "helsinki2" = "fi-hel2"
    "chicago"   = "us-chi1"
    "sanjose"   = "us-sjo1"
    "singapore" = "sg-sin1"
  }
}

variable "plans" {
  type = map
  default = {
    "5USD"  = "1xCPU-1GB"
    "10USD" = "1xCPU-2GB"
    "20USD" = "2xCPU-4GB"
  }
}

variable "storage_sizes" {
  type = map
  default = {
    "1xCPU-1GB" = "25"
    "1xCPU-2GB" = "50"
    "2xCPU-4GB" = "80"
  }
}

variable "templates" {
  type = map
  default = {
    "ubuntu18" = "01000000-0000-4000-8000-000030080200"
    "centos7"  = "01000000-0000-4000-8000-000050010300"
    "debian9"  = "01000000-0000-4000-8000-000020040100"
  }
}

variable "set_password" {
  type    = bool
  default = false
}

variable "users" {
  type    = list
  default = ["root", "user1", "user2"]
}

variable "plan" {
  type    = string
  default = "10USD"
}

variable "template" {
  type    = string
  default = "ubuntu18"
}
```

#### Using Input Variables

Input variables can be referenced in configurations:
```hcl
var.image_name
```

You can pass values through:
- **Files (`terraform.tfvars.json`)**:
  ```json
  {
    "private_key_path": "VALUE",
    "private_key_path": "VALUE"
  }
  ```
  Files named `terraform.tfvars`, `terraform.tfvars.json`, or ending with `.auto.tfvars/.auto.tfvars.json` are automatically loaded.

- **Command-line Flags**:
  ```bash
  terraform apply -var set_password="true"
  ```

- **Environment Variables**: Use the `TF_VAR_` prefix to define variables as environment variables:
  ```bash
  export TF_VAR_PASSWORD="password"
  ```
  Ensure the variable is declared in the configuration:
  ```hcl
  variable "PASSWORD" {
    default = ""
  }
  ```

### Output Variables

Output variables allow you to extract information from Terraform's state:
```hcl
output "public_ip" {
  value = upcloud_server.server_name.network_interface[0].ip_address
}
```

To access output variables, use:
```bash
terraform output {output_var_name}
```

### Local Values

Local values assign names to expressions for reuse within a module.

#### Declaring Local Values `locals.tf`

```hcl
locals {
  service_name = "forum"
  owner        = "Community Team"
  common_tags  = {
    Service = local.service_name
    Owner   = local.owner
  }
}
```

#### Using Local Values

Local values can be referenced as `local.<name>`:
```hcl
resource "aws_instance" "example" {
  tags = local.common_tags
}
```

#### Ephemeral Local Values (Terraform v1.10+)

Ephemeral local values depend on ephemeral inputs:
```hcl
variable "service_token" {
  type      = string
  ephemeral = true
}

locals {
  session_token = "${var.service_name}:${var.service_token}"
}
```

## Best Practices for Variables and Local Values

- Use input variables for configuration flexibility.
- Use local values to reduce repetition but avoid overuse for readability.
- Keep sensitive variables in environment variables or securely managed files.

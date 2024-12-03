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
9. [Variables and Local Values](#variables-and-local-values)

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
   
## Variables and Local Values

### Input Variables

- Input variables allow you to define values for your configuration that can be reused and customized.

#### Declaring Variables

Variables can be declared within the configuration or in a separate file like `variables.tf`:

```hcl
variable "private_key_path" {
  type    = string
  default = "/home/user/.ssh/terraform_rsa"
}

variable "public_key" {
  type    = string
  default = "ssh-rsa terraform_public_key"
}
```

You can also define more complex variables using maps or lists:

```hcl
variable "zones" {
  type = map
  default = {
    "amsterdam" = "nl-ams1"
    "london"    = "uk-lon1"
  }
}

variable "users" {
  type    = list
  default = ["root", "user1", "user2"]
}
```

#### Providing Values for Variables

- Use `terraform.tfvars.json` or `terraform.tfvars` files to define variable values:

Example `terraform.tfvars.json` file:
```json
{
  "image_name": "VALUE",
  "private_key_path": "new value"
}
```

You can also specify variable values during apply:
```bash
terraform apply -var 'set_password=true'
```

#### Using Environment Variables

Sensitive variables can be set as environment variables with the `TF_VAR_` prefix:
```bash
export TF_VAR_PASSWORD="password"
```

Ensure the variable is declared in `variables.tf`:
```hcl
variable "PASSWORD" {
  default = ""
}
```

### Output Variables

- Output variables let you extract and display information from your resources.
- Example declaration:
```hcl
output "public_ip" {
    value = upcloud_server.server_name.network_interface[0].ip_address
}
  ```
To access an output variable, use:
```bash
terraform output {output_var_name}
```

### Local Values

- Local values simplify configurations by defining reusable expressions.

#### Declaring Local Values
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
```hcl
resource "aws_instance" "example" {
  tags = local.common_tags
}
```

#### Ephemeral Values (Terraform v1.10+)
Ephemeral local values automatically derive their lifespan from ephemeral variables:

```hcl
variable "service_token" {
  type      = string
  ephemeral = true
}

locals {
  session_token = "${var.service_name}:${var.service_token}"
}
```

### Best Practices for Variables and Locals
- Use variables to parameterize your configuration.
- Use local values to avoid repetition, but keep configurations readable.

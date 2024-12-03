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

... (previous sections remain unchanged) ...

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

- You can also specify variable values during apply:
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
- To access an output variable, use:
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
